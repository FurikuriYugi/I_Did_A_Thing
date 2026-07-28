namespace ArgrillianThreat
{
	// -----------------------------
	// Shared Helper Classes
	// -----------------------------
	public static class ArgrillianGizmoHelpers
	{
		public static Command_Toggle Toggle(
			string label,
			string desc,
			System.Func<bool> isActive,
			System.Action toggleAction)
		{
			return new Command_Toggle
			{
				defaultLabel = label,
				defaultDesc = desc,
				isActive = isActive,
				toggleAction = toggleAction
			};
		}

		public static Pawn FindAlivePawnByThingID(Thing parent, int thingID)
		{
			if (thingID < 0) return null;
			if (parent?.Map == null) return null;

			// Use the alert system cache as the authority (no map-wide scanning).
			return ArgrillianAlertSystem.TryGetPatientFromCachedCall(parent.Map, thingID);
		}
	}

	public static class ArgrillianThreatMath
	{
		public static float ClampRadialRadius(float r) => Mathf.Min(r, 79.8f);
	}

	public static class ArgrillianThreatGeometry
	{
		private static readonly IntVec3[] Adj8 =
		{
			new IntVec3( 1,0, 0), new IntVec3(-1,0, 0),
			new IntVec3( 0,0, 1), new IntVec3( 0,0,-1),
			new IntVec3( 1,0, 1), new IntVec3( 1,0,-1),
			new IntVec3(-1,0, 1), new IntVec3(-1,0,-1)
		};

		public static IntVec3[] GetAdj8() => Adj8;
	}

	public static class ArgrillianThreatTuning
	{
		public const int PatientFightLockoutAfterRetreatTicks = 90;
	}

	public static class ArgillianThreatPatientTuning
	{
		private static float patientTooBadToEscortHPPercentThreshold = 0.65f; // patients below this get tended/escorted
		public static bool IsPatientTooBadToEscort(Pawn p)
		{
			if (p == null || p.Dead || p.health == null) return false;
			float hpPct = p.health.summaryHealth.SummaryHealthPercent;
			return p.health.summaryHealth != null && hpPct <= patientTooBadToEscortHPPercentThreshold;
		}

		public static bool JobIsMedicalForPatient(Job j, Pawn patient)
		{
			if (j == null || patient == null || j.def == null) return false;

			bool medical = j.def == JobDefOf.Rescue || j.def == JobDefOf.TendPatient;
			if (!medical) return false;

			return ArgillianThreatPatientTuning.GetPatientFromJob(j) == patient;
		}

		public static Pawn PatientAlreadyBeingTendedOrRescuedBySomeoneElse(Pawn medic, Pawn patient)
		{
			if (patient == null || patient.Dead) return null;
			if (medic == null) return null;
			if (patient.Map == null) return null;
			if (medic.Map == null) return null;
			if (medic.Map != patient.Map) return null;

			// If the medic is already doing medical on this exact patient, don't block it.
			if (medic.CurJob != null && JobIsMedicalForPatient(medic.CurJob, patient)) return null;

			int patientId = patient.thingIDNumber;

			// If there isn't an active cached call/entry for this patient, treat as not claimed.
			Pawn cachedPatient = ArgrillianAlertSystem.TryGetPatientFromCachedCall(medic.Map, patientId);
			if (cachedPatient == null) return null;

			// If this medic can reserve the hold right now, then nobody else is holding it.
			// This helper should not take the hold, so release immediately if we were able to reserve.
			bool reserved = ArgrillianAlertSystem.TryReserveMedicForPatient(medic, patient);
			if (reserved)
			{
				ArgrillianAlertSystem.ReleaseMedicHold(medic);
				return null;
			}

			// Otherwise, another medic is holding the patient in the alert system.
			return patient;
		}

		private static bool IsPatientServiceJob(JobDef def)
		{
			if (def == null) return false;

			// Core
			if (def == JobDefOf.TendPatient) return true;
			if (def == JobDefOf.Rescue) return true;

			// Moving injured pawns
			if (def == JobDefOf.HaulToCell) return true;
			if (def == JobDefOf.HaulToContainer) return true;

			// Many mods reuse/extend tending pipeline with other defs that still target the patient.
			// Add your custom job defs here if needed (paste their JobDef names and I’ll wire them in).
			// if (def.defName == "YourMod_YourCustomTendJob") return true;

			return false;
		}

		private static bool JobTargetsIncludePawn(Job job, Pawn pawn)
		{
			if (job == null || pawn == null) return false;

			// For jobs targeting the patient pawn directly:
			if (job.targetA == pawn) return true;
			if (job.targetB == pawn) return true;
			if (job.targetC == pawn) return true;

			// Some jobs use Thing targets (and pawns are Things). In that case, direct == may fail depending on context.
			// But targetA/targetB/targetC should still be Pawn objects if they point at the pawn.
			if (job.targetA.HasThing && job.targetA.Thing == pawn) return true;
			if (job.targetB.HasThing && job.targetB.Thing == pawn) return true;
			if (job.targetC.HasThing && job.targetC.Thing == pawn) return true;

			return false;
		}

		public static Pawn GetPatientFromJob(Job j)
		{
			if (j == null) return null;

			if (j.targetA.IsValid && j.targetA.Thing is Pawn a) return a;
			if (j.targetB.IsValid && j.targetB.Thing is Pawn b) return b;
			if (j.targetC.IsValid && j.targetC.Thing is Pawn c) return c;

			return null;
		}

		public static class PatientStabilityCache
		{
			private static readonly Dictionary<int, int> lastStableStartTick = new Dictionary<int, int>();
			private static readonly Dictionary<int, bool> lastWasStable = new Dictionary<int, bool>();

			private static int Now => Find.TickManager.TicksGame;

			private static bool IsDownedJobStable(Job job)
			{
				if (job == null) return false;
				if (job.def == JobDefOf.Wait) return true;
				if (job.def == JobDefOf.LayDown) return true;
				if (job.def == JobDefOf.Rescue) return true;
				if (job.def == JobDefOf.TendPatient) return true;

				string defName = job.def?.defName;
				if (string.IsNullOrEmpty(defName)) return false;

				defName = defName.ToLowerInvariant();
				if (defName.Contains("rescue")) return true;
				if (defName.Contains("lay")) return true;
				if (defName.Contains("bed")) return true;

				return false;
			}

			public static bool IsStableFor(Pawn patient, int requiredTicks)
			{
				if (patient == null || patient.Dead) return false;

				bool currentlyStable =
					patient.Downed && IsDownedJobStable(patient.CurJob) ||
					patient.pather == null ||
					!patient.pather.Moving;

				int id = patient.thingIDNumber;
				int now = Now;

				if (!lastWasStable.TryGetValue(id, out bool prevStable))
				{
					lastWasStable[id] = currentlyStable;
					if (currentlyStable)
						lastStableStartTick[id] = now;
					return currentlyStable && requiredTicks <= 0;
				}

				if (prevStable && !currentlyStable)
				{
					lastWasStable[id] = currentlyStable;
					lastStableStartTick.Remove(id);
					return false;
				}

				if (!prevStable && currentlyStable)
				{
					lastWasStable[id] = currentlyStable;
					lastStableStartTick[id] = now;
					return false;
				}

				if (!currentlyStable)
				{
					return false;
				}

				if (!lastStableStartTick.TryGetValue(id, out int startTick))
				{
					lastStableStartTick[id] = now;
					return false;
				}

				return (now - startTick) >= requiredTicks;
			}
		}
	}

	public static class ArgrillianGotoHelper
	{
		// Prevent changing Goto destinations too frequently even when your scoring changes slightly.
		public static class RepathCooldown
		{
			private struct LastRepath
			{
				public int tick;
				public IntVec3 cell;
			}

			private static readonly Dictionary<int, LastRepath> lastByPawnId = new Dictionary<int, LastRepath>();
			private static int lastNow = -1;

			public static int RepathMinIntervalTicks = 25;

			private static int Now => Find.TickManager.TicksGame;

			private static void ClearIfTimeWentBack()
			{
				int now = Now;
				if (lastNow != -1 && now < lastNow)
					lastByPawnId.Clear();
				lastNow = now;
			}

			public static bool ShouldAllowNewDestination(Pawn p, IntVec3 newCell)
			{
				if (p == null || p.Map == null) return true;
				ClearIfTimeWentBack();

				int id = p.thingIDNumber;
				if (!lastByPawnId.TryGetValue(id, out var lr))
					return true;

				if (lr.cell == newCell) return true;
				if ((Now - lr.tick) < RepathMinIntervalTicks) return false;

				return true;
			}

			public static void Mark(Pawn p, IntVec3 cell)
			{
				if (p == null || p.Map == null) return;
				ClearIfTimeWentBack();

				lastByPawnId[p.thingIDNumber] = new LastRepath
				{
					tick = Now,
					cell = cell
				};
			}
		}

		public static class GotoIssuedTickCache
		{
			private struct LastGoto
			{
				public int tick;
				public int cellX;
				public int cellY;
				public int cellZ;
			}

			private static readonly Dictionary<int, LastGoto> lastByPawnId = new Dictionary<int, LastGoto>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static bool WasIssuedThisTick(Pawn p, out IntVec3 lastCell)
			{
				lastCell = default;

				if (p == null) return false;
				if (p.Map == null) return false;

				int now = Now;
				if (lastNow != -1 && now < lastNow)
					lastByPawnId.Clear();
				lastNow = now;

				int id = p.thingIDNumber;
				if (!lastByPawnId.TryGetValue(id, out var lg))
					return false;

				if (lg.tick != now)
					return false;

				lastCell = new IntVec3(lg.cellX, lg.cellY, lg.cellZ);
				return true;
			}

			public static bool IsSameCellAsLastIssued(Pawn p, IntVec3 cell)
			{
				if (!WasIssuedThisTick(p, out IntVec3 lastCell))
					return false;

				return lastCell == cell;
			}

			public static void MarkIssued(Pawn p, IntVec3 cell)
			{
				if (p == null) return;

				int now = Now;
				if (lastNow != -1 && now < lastNow)
					lastByPawnId.Clear();
				lastNow = now;

				lastByPawnId[p.thingIDNumber] = new LastGoto
				{
					tick = now,
					cellX = cell.x,
					cellY = cell.y,
					cellZ = cell.z
				};
			}
		}

		// --- NEW: prevent Goto job churn / "Giggles started 10 jobs in one tick" ---
		public static Job KeepIfSameGoto(Pawn pawn, IntVec3 targetCell)
		{
			if (pawn?.CurJob == null) return null;
			if (pawn.CurJob.def != JobDefOf.Goto) return null;

			// For Goto, targetA is the cell.
			if (pawn.CurJob.targetA.Cell == targetCell)
				return pawn.CurJob;

			return null;
		}

		public static Job MakeGotoWithNoChurn(Pawn pawn, IntVec3 cell)
		{
			// repath cooldown: if we’re already going somewhere else, don’t change destination too quickly
			if (pawn?.CurJob != null && pawn.CurJob.def == JobDefOf.Goto)
			{
				IntVec3 current = pawn.CurJob.targetA.Cell;
				if (current != cell)
				{
					if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, cell))
						return pawn.CurJob;
				}
			}

			// Prevent “10 jobs in one tick” before CurJob updates.
			if (ArgrillianGotoHelper.GotoIssuedTickCache.IsSameCellAsLastIssued(pawn, cell))
			{
				var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, cell);
				if (keep != null) return keep;
				return null;
			}

			ArgrillianGotoHelper.GotoIssuedTickCache.MarkIssued(pawn, cell);
			ArgrillianGotoHelper.RepathCooldown.Mark(pawn, cell);
			return JobMaker.MakeJob(JobDefOf.Goto, cell);
		}
	}

	public static class ArgrillianThreatState
	{
		// --- JITTER FIX: combat commitment + repath cooldown ---
		public static class CombatCommit
		{
			private struct Commit
			{
				public int tick;
				public int hostileId;
			}

			private static readonly Dictionary<int, Commit> commitByPawnId = new Dictionary<int, Commit>();
			private static int lastNow = -1;

			public static int CommitTicks = 45;

			private static int Now => Find.TickManager.TicksGame;

			private static void ClearIfTimeWentBack()
			{
				int now = Now;
				if (lastNow != -1 && now < lastNow)
					commitByPawnId.Clear();
				lastNow = now;
			}

			public static void Mark(Pawn pursuer, Pawn hostile)
			{
				if (pursuer == null || hostile == null) return;
				ClearIfTimeWentBack();
				int now = Now;
				commitByPawnId[pursuer.thingIDNumber] = new Commit
				{
					tick = now,
					hostileId = hostile.thingIDNumber
				};
			}

			public static bool RecentlyCommitted(Pawn pursuer)
			{
				if (pursuer == null) return false;
				ClearIfTimeWentBack();

				if (!commitByPawnId.TryGetValue(pursuer.thingIDNumber, out var c))
					return false;

				return (Now - c.tick) < CommitTicks;
			}

			public static bool CommitMatchesHostile(Pawn pursuer, Pawn hostile)
			{
				if (pursuer == null || hostile == null) return false;
				ClearIfTimeWentBack();

				if (!commitByPawnId.TryGetValue(pursuer.thingIDNumber, out var c))
					return false;

				if ((Now - c.tick) >= CommitTicks) return false;
				return c.hostileId == hostile.thingIDNumber;
			}

			public static void Clear(Pawn pursuer)
			{
				if (pursuer == null) return;
				commitByPawnId.Remove(pursuer.thingIDNumber);
			}
		}

		// mode cache: 0 = fight, 1 = patient-retreat (goto safe cell)
		public static class ModeTickCache
		{
			private static readonly Dictionary<int, int> lastTick = new Dictionary<int, int>();
			private static readonly Dictionary<int, byte> lastMode = new Dictionary<int, byte>(); // 0 fight, 1 retreat
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static bool IsSameMode(Pawn p, byte mode)
			{
				if (p == null) return false;
				if (!lastMode.TryGetValue(p.thingIDNumber, out var m)) return false;
				return m == mode;
			}

			public static byte GetMode(Pawn p)
			{
				if (p == null) return 0;
				if (!lastMode.TryGetValue(p.thingIDNumber, out var m)) return 0;
				return m;
			}

			public static void MarkMode(Pawn p, byte mode)
			{
				if (p == null) return;
				int now = Now;
				if (lastNow != -1 && now < lastNow)
				{
					lastTick.Clear();
					lastMode.Clear();
				}
				lastNow = now;

				lastTick[p.thingIDNumber] = now;
				lastMode[p.thingIDNumber] = mode;
			}

			// Used for hysteresis-ish: don’t flip fight/retreat too quickly.
			public static bool CanSwitchMode(Pawn p, int cooldownTicks)
			{
				if (p == null) return true;
				int now = Now;

				if (lastNow != -1 && now < lastNow)
					lastTick.Clear();

				lastNow = now;

				if (!lastTick.TryGetValue(p.thingIDNumber, out int t))
					return true;

				return (now - t) >= cooldownTicks;
			}
		}

		public static class FightLockoutAfterRetreat
		{
			private static readonly Dictionary<int, int> lastRetreatEndTick = new Dictionary<int, int>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static void MarkRetreatEndAttempt(Pawn p)
			{
				if (p == null) return;

				int now = Now;

				// If ticks went backwards, reset tracking (avoid stale comparisons).
				if (lastNow != -1 && now < lastNow)
				{
					lastRetreatEndTick.Clear();
				}

				lastNow = now;
				lastRetreatEndTick[p.thingIDNumber] = now;
			}

			public static bool RecentlyEndedRetreat(Pawn p, int lockoutTicks)
			{
				if (p == null) return false;
				if (lockoutTicks <= 0) return false;

				int now = Now;

				// If ticks went backwards, do not apply stale lockouts.
				if (lastNow != -1 && now < lastNow)
				{
					lastRetreatEndTick.Clear();
					lastNow = now;
					return false;
				}

				lastNow = now;

				if (!lastRetreatEndTick.TryGetValue(p.thingIDNumber, out int t))
					return false;

				int delta = now - t;
				return delta <= lockoutTicks;
			}
		}

		// combat lock to prevent “hostile disappears for a tick -> return null -> hauling/eating/etc.”
		public static class CombatLock
		{
			private struct LockedTarget
			{
				public int hostileId;
				public int lastSeenTick;

				// PERF: store direct hostile pawn reference so we never do map-wide
				// pursuer.Map.mapPawns.AllPawnsSpawned scans just to resolve by thingID.
				public Pawn hostilePawn;
			}

			private static readonly Dictionary<int, LockedTarget> lockByPawnId = new Dictionary<int, LockedTarget>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static int HoldAfterHostileLostTicks = 120;

			public static void MarkSeen(Pawn pursuer, Pawn hostile)
			{
				if (pursuer == null || hostile == null) return;
				int now = Now;

				if (lastNow != -1 && now < lastNow)
					lockByPawnId.Clear();

				lastNow = now;

				lockByPawnId[pursuer.thingIDNumber] = new LockedTarget
				{
					hostileId = hostile.thingIDNumber,
					lastSeenTick = now,
					hostilePawn = hostile
				};
			}

			public static Pawn TryGetLockedHostile(Pawn pursuer)
			{
				if (pursuer == null) return null;
				if (pursuer.Map == null) return null;

				int id = pursuer.thingIDNumber;
				if (!lockByPawnId.TryGetValue(id, out var locked))
					return null;

				int now = Now;
				if (now - locked.lastSeenTick > HoldAfterHostileLostTicks)
					return null;

				// PERF: never scan map pawns; validate the cached reference.
				Pawn p = locked.hostilePawn;

				if (p == null) return null;
				if (p.Dead) return null;
				if (!p.Spawned) return null;
				if (pursuer.Map == null || p.Map != pursuer.Map) return null;
				if (p.thingIDNumber != locked.hostileId) return null;
				if (!p.HostileTo(pursuer)) return null;

				return p;
			}

			public static void Clear(Pawn pursuer)
			{
				if (pursuer == null) return;
				lockByPawnId.Remove(pursuer.thingIDNumber);
			}
		}

		public static class ThreatTickCache
		{
			private static readonly Dictionary<int, int> lastTick = new Dictionary<int, int>();
			private static int lastNow = -1;

			private static int Now => Find.TickManager.TicksGame;

			public static bool ShouldWait(Pawn p, int cooldown)
			{
				if (p == null) return true;

				int now = Now;

				if (lastNow != -1 && now < lastNow)
					lastTick.Clear();
				lastNow = now;

				int id = p.thingIDNumber;

				if (!lastTick.TryGetValue(id, out int t))
					return false;

				if (t > now)
					return false;

				return now - t < cooldown;
			}

			public static void MarkNow(Pawn p)
			{
				if (p == null) return;
				lastTick[p.thingIDNumber] = Now;
			}
		}

		// Shared awareness: last known hostile + tick when that awareness was obtained.
		// Used for "alerted / high alert" behaviors when a pawn didn't directly see the hostile.
		public static class AwarenessCache
		{
			private struct Awareness
			{
				public int hostileId;
				public IntVec3 lastKnownCell;
				public int lastSetTick;

				// 0 = from direct LOS/acquire, 1 = shared alert from ally
				public byte source;
			}

			public static void ClearIfHostileMatches(Pawn p, int hostileId)
			{
				if (p == null) return;
				if (hostileId < 0) return;

				if (!byPawnId.TryGetValue(p.thingIDNumber, out var a)) return;
				if (a.hostileId != hostileId) return;

				byPawnId.Remove(p.thingIDNumber);
			}

			private static readonly Dictionary<int, Awareness> byPawnId = new Dictionary<int, Awareness>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			// How long alerted pawns stay in "high alert"
			public static int HighAlertGraceTicks = 240;

			// After that, allow weaker "shared aware investigate" for this long
			public static int SharedInvestigateGraceTicks = 360;

			public static void Clear(Pawn p)
			{
				if (p == null) return;
				byPawnId.Remove(p.thingIDNumber);
			}

			// Call this when a pawn directly sees/acquires hostile.
			public static void MarkDirect(Pawn pawn, Pawn hostile)
			{
				if (pawn == null || hostile == null) return;
				MarkInternal(pawn, hostile, hostile.Position, source: 0);
			}

			// Call this when you want to share awareness to others.
			// lastKnownCell should be the source pawn's hostile position at the time.
			public static void MarkShared(Pawn pawn, IntVec3 lastKnownCell, int hostileId)
			{
				if (pawn == null) return;
				MarkInternal(pawn, null, lastKnownCell, source: 1, hostileId: hostileId);
			}

			private static void MarkInternal(Pawn pawn, Pawn hostileOrNull, IntVec3 lastKnownCell, byte source, int hostileId = -1)
			{
				if (pawn == null) return;

				int now = Now;
				if (lastNow != -1 && now < lastNow)
					byPawnId.Clear();
				lastNow = now;

				int hid = hostileId;
				if (hid < 0 && hostileOrNull != null)
					hid = hostileOrNull.thingIDNumber;

				byPawnId[pawn.thingIDNumber] = new Awareness
				{
					hostileId = hid,
					lastKnownCell = lastKnownCell,
					lastSetTick = now,
					source = source
				};
			}

			public static bool HasAnyAwareness(Pawn p)
			{
				if (p == null) return false;
				return byPawnId.ContainsKey(p.thingIDNumber);
			}

			public static bool IsHighAlert(Pawn p)
			{
				if (p == null) return false;
				if (!byPawnId.TryGetValue(p.thingIDNumber, out var a)) return false;

				int now = Now;
				if (now - a.lastSetTick > HighAlertGraceTicks)
					return false;

				return true;
			}

			public static bool IsSharedInvestigate(Pawn p)
			{
				if (p == null) return false;
				if (!byPawnId.TryGetValue(p.thingIDNumber, out var a)) return false;

				int now = Now;
				if (now - a.lastSetTick > SharedInvestigateGraceTicks)
					return false;

				// If direct LOS, treat it as high alert anyway; otherwise shared investigate.
				return a.source == 1;
			}

			public static IntVec3 GetLastKnownCell(Pawn p)
			{
				if (p == null) return default;
				if (!byPawnId.TryGetValue(p.thingIDNumber, out var a)) return default;
				return a.lastKnownCell;
			}

			public static int GetLastKnownHostileId(Pawn p)
			{
				if (p == null) return -1;
				if (!byPawnId.TryGetValue(p.thingIDNumber, out var a)) return -1;
				return a.hostileId;
			}
		}

		// Tracks how long a pawn has been outside the melee band while trying to fight.
		public static class MeleeKiteAbortTickCache
		{
			private static readonly System.Collections.Generic.Dictionary<Pawn, int> OutOfBandStartTick
				= new System.Collections.Generic.Dictionary<Pawn, int>();

			public static void Reset(Pawn pawn)
			{
				if (pawn == null) return;
				OutOfBandStartTick.Remove(pawn);
			}

			public static bool IsOutOfBandLongEnough(Pawn pawn, bool currentlyOutOfBand, int requiredTicks, int currentTick, out int outOfBandTicks)
			{
				outOfBandTicks = 0;
				if (pawn == null) return false;

				if (!currentlyOutOfBand)
				{
					OutOfBandStartTick.Remove(pawn);
					return false;
				}

				if (!OutOfBandStartTick.TryGetValue(pawn, out int startTick))
				{
					OutOfBandStartTick[pawn] = currentTick;
					startTick = currentTick;
				}

				outOfBandTicks = currentTick - startTick;
				return outOfBandTicks >= requiredTicks;
			}
		}
	}

	public static class ArgrillianMedicalState
	{
		public static class MedicTendTaskStickiness
		{
			private static readonly Dictionary<int, int> lastTendTaskTick = new Dictionary<int, int>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static void MarkTask(Pawn medic)
			{
				if (medic == null) return;

				int now = Now;
				if (lastNow != -1 && now < lastNow)
					lastTendTaskTick.Clear();
				lastNow = now;

				lastTendTaskTick[medic.thingIDNumber] = now;
			}

			public static bool RecentlyTookTendTask(Pawn medic, int stickinessTicks)
			{
				if (medic == null) return false;

				int now = Now;
				if (lastNow != -1 && now < lastNow)
					lastTendTaskTick.Clear();
				lastNow = now;

				if (!lastTendTaskTick.TryGetValue(medic.thingIDNumber, out int t))
					return false;

				return (now - t) <= stickinessTicks;
			}
		}

		public static class MedicTickCache
		{
			private static readonly Dictionary<int, int> lastTick = new Dictionary<int, int>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			public static bool ShouldWait(Pawn p, int cooldown)
			{
				if (p == null) return true;

				int now = Now;
				if (lastNow != -1 && now < lastNow) lastTick.Clear();
				lastNow = now;

				int id = p.thingIDNumber;
				if (!lastTick.TryGetValue(id, out int t)) return false;
				if (t > now) return false;

				return now - t < cooldown;
			}

			public static void MarkNow(Pawn p)
			{
				if (p == null) return;
				lastTick[p.thingIDNumber] = Now;
			}
		}

		public static class RescueAttemptCooldownCache
		{
			private static readonly Dictionary<int, int> lastAttemptTick = new Dictionary<int, int>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			private static int Key(Pawn medic, Pawn patient)
			{
				unchecked
				{
					return (medic.thingIDNumber * 397) ^ patient.thingIDNumber;
				}
			}

			public static bool RecentlyAttempted(Pawn medic, Pawn patient, int cooldownTicks)
			{
				if (medic == null || patient == null) return false;

				int now = Now;
				if (lastNow != -1 && now < lastNow) lastAttemptTick.Clear();
				lastNow = now;

				int k = Key(medic, patient);
				if (!lastAttemptTick.TryGetValue(k, out int t))
					return false;

				if (t > now) return false;
				return (now - t) < cooldownTicks;
			}

			public static void MarkAttempt(Pawn medic, Pawn patient)
			{
				if (medic == null || patient == null) return;
				lastAttemptTick[Key(medic, patient)] = Now;
			}
		}

		public static class RescueCommitCooldownCache
		{
			private static readonly Dictionary<int, int> lastCommitTick = new Dictionary<int, int>();
			private static int lastNow = -1;
			private static int Now => Find.TickManager.TicksGame;

			private static int Key(Pawn medic, Pawn patient)
			{
				unchecked
				{
					return (medic.thingIDNumber * 397) ^ patient.thingIDNumber;
				}
			}

			public static bool RecentlyCommitted(Pawn medic, Pawn patient, int commitTicks)
			{
				if (medic == null || patient == null) return false;

				int now = Now;
				if (lastNow != -1 && now < lastNow) lastCommitTick.Clear();
				lastNow = now;

				int k = Key(medic, patient);
				if (!lastCommitTick.TryGetValue(k, out int t))
					return false;

				if (t > now) return false;
				return (now - t) <= commitTicks;
			}

			public static void MarkCommit(Pawn medic, Pawn patient)
			{
				if (medic == null || patient == null) return;
				lastCommitTick[Key(medic, patient)] = Now;
			}
		}

		public static class PatientMedicHold
		{
			// Medic thingID -> cached patientId (int only; Pawn resolved via ArgrillianAlertSystem lookup)
			private static readonly Dictionary<int, int> patientIdByMedic = new Dictionary<int, int>();

			public static Pawn GetHeldPatient(Pawn medic)
			{
				if (medic == null) return null;
				if (medic.Map == null) return null;
				if (!medic.Spawned) return null;
				if (medic.Dead) return null;

				int medicId = medic.thingIDNumber;
				if (!patientIdByMedic.TryGetValue(medicId, out int patientId))
					return null;

				// Resolve from alert cache (no map scans).
				return ArgrillianAlertSystem.TryGetPatientFromCachedCall(medic.Map, patientId);
			}

			public static void Lock(Pawn medic, Pawn patient)
			{
				if (medic == null || patient == null) return;
				if (medic.Dead || patient.Dead) return;
				if (medic.Map == null || patient.Map == null) return;
				if (medic.Map != patient.Map) return;
				if (!medic.Spawned || !patient.Spawned) return;

				int medicId = medic.thingIDNumber;
				int patientId = patient.thingIDNumber;

				// Enforce exclusivity inside alert system.
				bool accepted = ArgrillianAlertSystem.TryReserveMedicForPatient(medic, patient);
				if (!accepted) return;

				patientIdByMedic[medicId] = patientId;
			}

			// Release by medic only; resolves nothing; no scanning.
			public static void ReleaseForMedic(Pawn medic)
			{
				if (medic == null) return;

				int medicId = medic.thingIDNumber;
				patientIdByMedic.Remove(medicId);

				ArgrillianAlertSystem.ReleaseMedicHold(medic);

				// Preserve your existing “stop drift” behavior without clearing queued jobs.
				Pawn patient = GetHeldPatient(medic);
				if (patient != null && !patient.Dead)
					patient.pather?.StopDead();
			}
		}
	}

	// NEW: Event Driven Alert System
	public static class ArgrillianAlertSystem
	{
		private sealed class MapCache
		{
			public readonly List<Pawn> recipients = new List<Pawn>(64);

			// Used to prevent duplicates in recipients.
			public readonly HashSet<int> recipientIds = new HashSet<int>();

			// Soft TTL cleanup cadence for recipients list pruning.
			public int lastRecipientPruneTick = -1;
		}

		private static readonly Dictionary<int, MapCache> byMapId = new Dictionary<int, MapCache>();

		// Prune recipients list at most this often per map.
		// Keeps this event-driven without doing work every broadcast.
		private const int RecipientPruneIntervalTicks = 60;

		private static MapCache GetOrCreate(Map map)
		{
			if (map == null) return null;

			int mapId = map.uniqueID;
			if (!byMapId.TryGetValue(mapId, out MapCache cache))
			{
				cache = new MapCache();
				byMapId[mapId] = cache;
			}

			return cache;
		}

		private static void PruneRecipients(MapCache cache, Map map)
		{
			if (cache == null || map == null) return;

			int now = Find.TickManager.TicksGame;
			if (cache.lastRecipientPruneTick >= 0 &&
				now - cache.lastRecipientPruneTick < RecipientPruneIntervalTicks)
				return;

			cache.lastRecipientPruneTick = now;

			for (int i = cache.recipients.Count - 1; i >= 0; i--)
			{
				Pawn p = cache.recipients[i];

				bool remove =
					p == null ||
					p.Dead ||
					!p.Spawned ||
					p.Map != map;

				if (remove)
				{
					int id = p != null ? p.thingIDNumber : -1;
					if (id >= 0) cache.recipientIds.Remove(id);
					cache.recipients.RemoveAt(i);
				}
			}
		}

		/// <summary>
		/// Call this once for any pawn that should RECEIVE alerts (radio recipients).
		/// Typical usage: in the comp's SpawnSetup / OnEnable for pawns you want participating.
		/// </summary>
		public static void RegisterRecipient(Pawn pawn)
		{
			if (pawn == null) return;
			if (pawn.Dead) return;
			if (pawn.Map == null) return;
			if (!pawn.Spawned) return;

			MapCache cache = GetOrCreate(pawn.Map);
			if (cache == null) return;

			int id = pawn.thingIDNumber;
			if (id < 0) return;

			if (cache.recipientIds.Add(id))
				cache.recipients.Add(pawn);
		}

		/// <summary>
		/// Optional: call when a pawn should stop receiving alerts (despawn/death/comp disabled).
		/// </summary>
		public static void UnregisterRecipient(Pawn pawn)
		{
			if (pawn == null) return;
			if (pawn.Map == null) return;

			if (!byMapId.TryGetValue(pawn.Map.uniqueID, out MapCache cache))
				return;

			int id = pawn.thingIDNumber;
			if (id < 0) return;

			if (!cache.recipientIds.Remove(id))
				return;

			// Remove from recipients list (rare; keeps Register fast).
			for (int i = cache.recipients.Count - 1; i >= 0; i--)
			{
				Pawn p = cache.recipients[i];
				if (p == null || p.thingIDNumber == id)
					cache.recipients.RemoveAt(i);
			}
		}

		/// <summary>
		/// Event-driven broadcast dispatcher.
		/// No map scanning: only not-expired registered recipients are consulted.
		/// </summary>
		public static void BroadcastSharedAwareness(
			Pawn source,
			Pawn hostile,
			float allyRadius,
			bool squadOnly)
		{
			if (source == null) return;
			if (hostile == null) return;
			if (source.Map == null) return;
			if (hostile.Dead || !hostile.Spawned) return;
			if (allyRadius < 0f) allyRadius = 0f;

			MapCache cache = GetOrCreate(source.Map);
			if (cache == null) return;

			PruneRecipients(cache, source.Map);

			IntVec3 lastKnown = hostile.Position;
			int hostileId = hostile.thingIDNumber;

			// Recipients are already cached; we only filter per broadcast.
			for (int i = 0; i < cache.recipients.Count; i++)
			{
				Pawn p = cache.recipients[i];
				if (p == null) continue;
				if (p.Dead || !p.Spawned) continue;
				if (p.Map != source.Map) continue;
				if (p == source) continue;

				if (p.Faction != source.Faction) continue;

				if (squadOnly)
				{
					// If this comp doesn't exist, treat as not in squad channel.
					CompArgrillianThreatSettings ts = p.GetComp<CompArgrillianThreatSettings>();
					if (ts == null || ts.squadMode != true) continue;
				}

				if (p.Position.DistanceTo(source.Position) > allyRadius) continue;

				// Keep your existing awareness cache semantics intact.
				if (ArgrillianThreatState.AwarenessCache.IsHighAlert(p) ||
					ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(p))
					continue;

				ArgrillianThreatState.AwarenessCache.MarkShared(p, lastKnown, hostileId);
			}
		}

		// ----------------------------
		// PatientCall (event-driven)
		// ----------------------------

		private enum PatientCallSeverity : byte
		{
			Downed = 1,
			Bleed = 2
		}

		private sealed class PatientCallEntry
		{
			public int patientId;
			public int patientThingIdNumber;
			public Pawn patient;

			public PatientCallSeverity severity;

			// For “event freshness” / coalescing:
			// - if we receive another call for the same patientThingIdNumber, we refresh TTL
			// - we only upgrade severity (Bleed > Downed)
			public int lastUpdateTick;

			// TTL is enforced via expiryTick derived from lastUpdateTick each cleanup.
			public int expiryTick;
		}

		// Map-level patient call cache (keyed by patient thingIDNumber)
		private sealed class PatientMapCache
		{
			public readonly Dictionary<int, PatientCallEntry> byPatientId = new Dictionary<int, PatientCallEntry>();
			public int lastPatientPruneTick = -1;
		}

		private static readonly Dictionary<int, PatientMapCache> byPatientMapId = new Dictionary<int, PatientMapCache>();

		// Soft cleanup cadence + TTL. Tune as desired.
		private const int PatientCallPruneIntervalTicks = 60;
		private const int PatientCallTTL_DownedTicks = 250;
		private const int PatientCallTTL_BleedTicks = 400;

		private static PatientMapCache GetOrCreatePatientCache(Map map)
		{
			if (map == null) return null;

			int mapId = map.uniqueID;
			if (!byPatientMapId.TryGetValue(mapId, out PatientMapCache cache))
			{
				cache = new PatientMapCache();
				byPatientMapId[mapId] = cache;
			}

			return cache;
		}

		public static bool IsPatientHeldByAnyMedic(Map map, int patientId)
		{
			if (map == null) return false;
			if (patientId < 0) return false;

			// PatientCall cache pruning happens inside TryGetPatientCallEntry,
			// but held-map pruning is lightweight and only done on updates elsewhere.
			// Here we only need the current reservation state.
			return medicIdByPatientId.TryGetValue(patientId, out int _);
		}

		private static PatientCallSeverity ComputePatientSeverity(Pawn patient)
		{
			if (patient == null) return PatientCallSeverity.Downed;

			// Severity ordering locked: Bleed > downed
			// “Bleed” = has BloodLoss hediff
			// (Using the same HasBloodLossStatic/BloodLossSeverityStatic approach you already have in this file.)
			if (patient.health?.hediffSet != null && patient.health.hediffSet.HasHediff(HediffDefOf.BloodLoss))
				return PatientCallSeverity.Bleed;

			if (patient.Downed)
				return PatientCallSeverity.Downed;

			// If we get called for a non-downed/non-bleeding state, treat as downed-tier so it expires quickly.
			return PatientCallSeverity.Downed;
		}

		// Prune expired / invalid entries.
		private static void PrunePatientCalls(PatientMapCache cache, Map map)
		{
			if (cache == null || map == null) return;

			int now = Find.TickManager.TicksGame;
			if (cache.lastPatientPruneTick >= 0 && now - cache.lastPatientPruneTick < PatientCallPruneIntervalTicks)
				return;

			cache.lastPatientPruneTick = now;

			// Remove invalid/expired
			List<int> toRemove = null;

			foreach (var kv in cache.byPatientId)
			{
				int id = kv.Key;
				PatientCallEntry e = kv.Value;
				if (e == null)
				{
					if (toRemove == null) toRemove = new List<int>(4);
					toRemove.Add(id);
					continue;
				}

				Pawn p = e.patient;
				bool invalid =
					p == null ||
					p.Dead ||
					!p.Spawned ||
					p.Map != map;

				bool expired = now >= e.expiryTick;

				if (invalid || expired)
				{
					if (toRemove == null) toRemove = new List<int>(4);
					toRemove.Add(id);
				}
			}

			if (toRemove == null) return;

			for (int i = 0; i < toRemove.Count; i++)
			{
				int id = toRemove[i];
				cache.byPatientId.Remove(id);
			}
		}

		// Publish/refresh a PatientCall (coalesced by patient thingIDNumber).
		// “call trigger” locked:
		// - self-state: when a pawn gets injured/down
		// - observer-state: when any pawn sees injured/down
		//
		// This method is the single entry point you’ll call from those triggers.
		public static void PublishPatientCall(Pawn callerOrObserver, Pawn patient, bool wasDownedOrBleedingNow)
		{
			if (patient == null) return;
			if (patient.Dead) return;
			if (patient.Map == null) return;
			if (!patient.Spawned) return;

			// Only create/update if the pawn is actually downed or bleeding (or caller said it was).
			PatientCallSeverity newSev = ComputePatientSeverity(patient);

			if (!wasDownedOrBleedingNow && newSev != PatientCallSeverity.Downed)
			{
				// If caller doesn't assert relevance and severity came out as downed-tier,
				// you can tighten this later; for now keep it simple and allow downed-tier only.
			}

			PatientMapCache cache = GetOrCreatePatientCache(patient.Map);
			if (cache == null) return;

			// Keyed by patient thingIDNumber for coalescing.
			int patientId = patient.thingIDNumber;
			if (patientId < 0) return;

			PrunePatientCalls(cache, patient.Map);

			int now = Find.TickManager.TicksGame;

			PatientCallEntry entry;
			if (!cache.byPatientId.TryGetValue(patientId, out entry) || entry == null)
			{
				entry = new PatientCallEntry();
				entry.patientId = patientId;
				entry.patientThingIdNumber = patientId;
				entry.patient = patient;
				entry.lastUpdateTick = now;
				entry.severity = newSev;

				int ttl = (newSev == PatientCallSeverity.Bleed) ? PatientCallTTL_BleedTicks : PatientCallTTL_DownedTicks;
				entry.expiryTick = now + ttl;

				cache.byPatientId[patientId] = entry;
				return;
			}

			// Refresh TTL + patient reference
			entry.patient = patient;
			entry.lastUpdateTick = now;

			// Upgrade severity if needed (Bleed > downed)
			if (newSev > entry.severity)
				entry.severity = newSev;

			int effectiveTtl = (entry.severity == PatientCallSeverity.Bleed) ? PatientCallTTL_BleedTicks : PatientCallTTL_DownedTicks;
			entry.expiryTick = now + effectiveTtl;
		}

		// Query: pick the best patient for a medic from cached calls only.
		// Severity ordering locked: Bleed > downed
		// heldPatient exclusivity will be handled in JobGiver (next step).
		public static Pawn GetBestPatientFromCalls(Pawn medic, float searchRadius)
		{
			if (medic == null) return null;
			if (medic.Map == null) return null;
			if (!medic.Spawned) return null;

			PatientMapCache cache = GetOrCreatePatientCache(medic.Map);
			if (cache == null) return null;

			PrunePatientCalls(cache, medic.Map);

			float r = Mathf.Max(0f, searchRadius);
			//float bestScore = float.NegativeInfinity;
			Pawn best = null;

			// Two-pass selection makes severity ordering strict:
			// first pick Bleed, then Downed.
			Pawn bestBleed = null;
			float bestBleedScore = float.NegativeInfinity;

			Pawn bestDowned = null;
			float bestDownedScore = float.NegativeInfinity;

			foreach (var kv in cache.byPatientId)
			{
				PatientCallEntry e = kv.Value;
				if (e == null) continue;

				Pawn p = e.patient;
				if (p == null) continue;
				if (p.Dead || !p.Spawned) continue;
				if (p.Map != medic.Map) continue;

				float dist = medic.Position.DistanceTo(p.Position);
				if (dist > r) continue;

				// Score: severity first, then urgency by HP% / downed
				float hpPct = p.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
				float hpUrgency = (1f - hpPct) * 300f;

				float bleedBonus = (e.severity == PatientCallSeverity.Bleed) ? 250f : 0f;
				float downedBonus = (p.Downed) ? 50f : 0f;

				float score = hpUrgency + bleedBonus + downedBonus;

				if (e.severity == PatientCallSeverity.Bleed)
				{
					if (score > bestBleedScore)
					{
						bestBleedScore = score;
						bestBleed = p;
					}
				}
				else
				{
					if (score > bestDownedScore)
					{
						bestDownedScore = score;
						bestDowned = p;
					}
				}
			}

			// Locked ordering: Bleed > downed
			if (bestBleed != null) best = bestBleed;
			else best = bestDowned;

			return best;
		}

		// ----------------------------
		// PatientHold / exclusivity integration (no map scanning)
		// ----------------------------

		// patientId -> medicId (who is currently reserved to tend this patient)
		private static readonly Dictionary<int, int> medicIdByPatientId = new Dictionary<int, int>();
		// medicId -> patientId (what patient this medic is currently holding)
		private static readonly Dictionary<int, int> patientIdByMedicId = new Dictionary<int, int>();

		private static PatientCallEntry TryGetPatientCallEntry(Map map, int patientId)
		{
			if (map == null) return null;
			if (patientId < 0) return null;

			PatientMapCache cache = GetOrCreatePatientCache(map);
			if (cache == null) return null;

			PrunePatientCalls(cache, map);

			if (!cache.byPatientId.TryGetValue(patientId, out PatientCallEntry e) || e == null)
				return null;

			// Double-check entry still maps to a valid spawned pawn on this map.
			Pawn p = e.patient;
			if (p == null) return null;
			if (p.Dead || !p.Spawned) return null;
			if (p.Map == null || p.Map != map) return null;
			if (p.thingIDNumber != patientId) return null;

			return e;
		}

		// Resolve a patient Pawn from the cached PatientCall entry (no map.spawnedThings iteration).
		public static Pawn TryGetPatientFromCachedCall(Map map, int patientId)
		{
			PatientCallEntry e = TryGetPatientCallEntry(map, patientId);
			return e != null ? e.patient : null;
		}

		// Enforce: at most one medic holds a given patient at a time.
		// Accepts reservation if:
		// - patient is in the cached PatientCall set, AND
		// - nobody else currently holds that same patient.
		//
		// Returns true if accepted; false otherwise.
		public static bool TryReserveMedicForPatient(Pawn medic, Pawn patient)
		{
			if (medic == null || patient == null) return false;
			if (medic.Dead || patient.Dead) return false;
			if (medic.Map == null || patient.Map == null) return false;
			if (medic.Map != patient.Map) return false;
			if (!medic.Spawned || !patient.Spawned) return false;

			int medicId = medic.thingIDNumber;
			int patientId = patient.thingIDNumber;

			// Ensure the patient is one of the active cached calls.
			Pawn cached = TryGetPatientFromCachedCall(medic.Map, patientId);
			if (cached == null) return false;

			// If some medic already holds this patient, reject unless it's the same medic.
			if (medicIdByPatientId.TryGetValue(patientId, out int existingMedicId))
			{
				if (existingMedicId == medicId)
					return true;

				return false;
			}

			// If this medic already holds a different patient, we allow overwrite semantics
			// only if caller releases first. Here we simply reject to keep behavior explicit.
			if (patientIdByMedicId.TryGetValue(medicId, out int existingPatientId))
			{
				if (existingPatientId != patientId)
					return false;
			}

			medicIdByPatientId[patientId] = medicId;
			patientIdByMedicId[medicId] = patientId;
			return true;
		}

		public static void ReleaseMedicHold(Pawn medic)
		{
			if (medic == null) return;
			if (medic.Dead) return;
			if (medic.Map == null) return;
			if (!medic.Spawned) return;

			int medicId = medic.thingIDNumber;
			if (!patientIdByMedicId.TryGetValue(medicId, out int patientId))
				return;

			patientIdByMedicId.Remove(medicId);

			// Only remove if it still points at this medic.
			if (medicIdByPatientId.TryGetValue(patientId, out int heldMedicId) && heldMedicId == medicId)
				medicIdByPatientId.Remove(patientId);
		}

		// ----------------------------
		// PatientCall transition triggers (event publisher glue)
		// ----------------------------
		// These helpers are designed to be called from the existing pawn tick/state update
		// where you already evaluate down/bleed for medic/patient logic.
		//
		// They are transition-based (coalesced): we only publish when a pawn *enters*
		// downed/bleeding state, not every tick.
		private static readonly Dictionary<int, byte> patientDownBleedStateByPawnId = new Dictionary<int, byte>();

		// bit0 = downed, bit1 = bleeding
		private static byte ComputeDownBleedState(Pawn p)
		{
			if (p == null || p.Dead) return 0;
			if (!p.Spawned || p.Map == null) return 0;

			byte s = 0;
			if (p.Downed) s |= 1;

			bool bleeding =
				p.health?.hediffSet != null
				&& p.health.hediffSet.HasHediff(HediffDefOf.BloodLoss);

			if (bleeding) s |= 2;
			return s;
		}

		// Call this from the pawn's "self-state" evaluation (once per relevant update, e.g., Think/CompTick).
		// It will publish PatientCall on transition into downed or bleeding.
		public static void NotifyPawnSelfState(Pawn pawn)
		{
			if (pawn == null) return;
			if (pawn.Dead) return;

			int id = pawn.thingIDNumber;
			if (id < 0) return;

			byte prev = 0;
			patientDownBleedStateByPawnId.TryGetValue(id, out prev);

			byte cur = ComputeDownBleedState(pawn);
			if (cur == prev) return;

			// Transition into downed/bleeding should publish immediately.
			bool enteredDowned = ((cur & 1) != 0) && ((prev & 1) == 0);
			bool enteredBleeding = ((cur & 2) != 0) && ((prev & 2) == 0);

			if (enteredDowned || enteredBleeding)
			{
				bool wasDownedOrBleedingNow = enteredDowned || enteredBleeding;
				PublishPatientCall(pawn, pawn, wasDownedOrBleedingNow);
			}

			// Keep latest state so we don't churn.
			patientDownBleedStateByPawnId[id] = cur;
		}

		// Call this from the observer-state evaluation where you already know "observer sees target downed/bleeding".
		// This version does NOT require the observer to track previous state; it just publishes as a call.
		public static void NotifyObserverSeesInjury(Pawn observer, Pawn target)
		{
			if (target == null) return;
			if (target.Dead) return;
			if (target.Downed == false && (target.health?.hediffSet == null || !target.health.hediffSet.HasHediff(HediffDefOf.BloodLoss)))
				return;

			bool wasDownedOrBleedingNow = true;
			PublishPatientCall(observer, target, wasDownedOrBleedingNow);
		}

		// ----------------------------
		// HostileCall transition triggers (event publisher glue)
		// ----------------------------
		// We keep this edge/transition based to avoid per-tick spam.
		//
		// Hostile call lifecycle:
		// - Seen: publish/refresh a hostile call with last-known cell.
		// - Lost sight: if still alive, switch state to "investigate" with last-known cell.
		// - Eliminated/left map: invalidate call so recipients stop pursuing stale info.

		private sealed class HostileCall
		{
			public int callerPawnId = -1;
			public int hostileThingId = -1;
			public int mapId = -1;

			public IntVec3 lastKnownCell;
			public bool hasLOS = false;

			// If true: recipients should investigate lastKnownCell rather than treat as confirmed pursuit.
			public bool investigateMode = false;

			public int lastUpdateTick = -1;
		}

		// mapId -> hostileThingId -> call
		private static readonly Dictionary<int, Dictionary<int, HostileCall>> hostileCallsByMap = new Dictionary<int, Dictionary<int, HostileCall>>();

		// (callerId, hostileId) -> lastPublishedStateBit (edge coalescing)
		// bit0: hadLOS=true/false at last publish
		// bit1: eliminated/invalidated published
		private static readonly Dictionary<int, byte> hostileSeenLostByCallerAndHostile = new Dictionary<int, byte>();

		private static int HostileEdgeKey(int callerId, int hostileId)
		{
			unchecked
			{
				return (callerId * 397) ^ hostileId;
			}
		}

		private static Dictionary<int, HostileCall> GetHostileCallsForMap(Map map)
		{
			if (map == null) return null;

			int mapId = map.uniqueID;
			if (!hostileCallsByMap.TryGetValue(mapId, out var dict))
			{
				dict = new Dictionary<int, HostileCall>();
				hostileCallsByMap[mapId] = dict;
			}

			return dict;
		}

		private static HostileCall GetOrCreateHostileCall(Map map, Pawn caller, Pawn hostile)
		{
			if (map == null || caller == null || hostile == null) return null;

			var dict = GetHostileCallsForMap(map);
			if (dict == null) return null;

			int hostileId = hostile.thingIDNumber;
			if (!dict.TryGetValue(hostileId, out HostileCall call))
			{
				call = new HostileCall
				{
					callerPawnId = caller.thingIDNumber,
					hostileThingId = hostileId,
					mapId = map.uniqueID,
					lastKnownCell = hostile.Position,
					hasLOS = false,
					investigateMode = false,
					lastUpdateTick = Find.TickManager.TicksGame
				};
				dict[hostileId] = call;
			}

			return call;
		}

		// Prune/cancel stale calls so recipients don’t chase forever.
		private const int HostileCallTtlTicks = 1500; // tuned conservatively; adjust if you want faster forget.

		private static void PruneHostileCallsIfNeeded(Map map)
		{
			if (map == null) return;
			var dict = GetHostileCallsForMap(map);
			if (dict == null || dict.Count == 0) return;

			int now = Find.TickManager.TicksGame;

			HostileThingPruneBuffer.Clear();
			foreach (var kvp in dict)
			{
				var call = kvp.Value;
				if (call == null) continue;
				if (call.lastUpdateTick < 0) continue;

				if (now - call.lastUpdateTick > HostileCallTtlTicks)
					HostileThingPruneBuffer.Add(kvp.Key);
			}

			for (int i = 0; i < HostileThingPruneBuffer.Count; i++)
				dict.Remove(HostileThingPruneBuffer[i]);
		}

		private static readonly List<int> HostileThingPruneBuffer = new List<int>(64);

		// Recipients mode routing:
		// - If hasLOS: recipients should respond as "combat/intercept" (not implement new job logic here;
		//   they already have ExecuteAlertedMode gated by AwarenessCache flags)
		// - If lost sight: recipients should switch to "shared investigate" mode
		private static void PublishHostileCallToRecipients(Map map, IntVec3 lastKnown, Pawn hostile)
		{
			var cache = GetOrCreate(map);
			if (cache == null) return;

			// Decide "investigate mode" based on caller/call state stored in hostileCallsByMap.
			// We route by setting AwarenessCache flags (which your job logic already reads).
			//
			// NOTE: We do NOT do map-wide scanning; recipients are the registered list only.
			PruneRecipients(cache, map);

			// Determine which recipients to notify:
			// - pawns not in battle should investigate
			// - combat pawns can respond based on your existing ExecuteAlertedMode/hostile awareness logic
			int hostileId = hostile.thingIDNumber;
			for (int i = cache.recipients.Count - 1; i >= 0; i--)
			{
				Pawn recipient = cache.recipients[i];
				if (recipient == null) continue;
				if (recipient.Dead) continue;
				if (!recipient.Spawned) continue;
				if (recipient.Map != map) continue;

				// If you already have an "investigate" flag setter, use it here.
				// Otherwise, your next step is to implement those setters in AwarenessCache.
				//
				// We’ll use the existing cache methods conceptually by calling the same dispatcher your patient calls use.
				// (If this repo’s AwarenessCache naming differs, you’ll point me to the exact methods and I’ll patch.)
				ArgrillianThreatState.AwarenessCache.MarkShared(recipient, lastKnown, hostileId);
			}
		}

		// PUBLIC API (call from pawn edge transitions)
		public static void NotifyPawnSeesHostile(Pawn caller, Pawn hostile, IntVec3 lastKnownCell)
		{
			if (caller == null || hostile == null) return;
			if (caller.Dead || hostile.Dead) return;
			if (caller.Map == null || hostile.Map == null) return;
			if (caller.Map != hostile.Map) return;

			Map map = caller.Map;
			int now = Find.TickManager.TicksGame;

			PruneHostileCallsIfNeeded(map);

			int callerId = caller.thingIDNumber;
			int hostileId = hostile.thingIDNumber;
			int edgeKey = HostileEdgeKey(callerId, hostileId);

			byte prev = 0;
			hostileSeenLostByCallerAndHostile.TryGetValue(edgeKey, out prev);

			// Only publish on edge into "LOS confirmed" (bit0 transition).
			byte cur = (byte)(prev | 1);

			if ((prev & 1) == 1)
			{
				// Already in LOS-confirmed state for this caller/hostile: just refresh TTL + location coalesced.
				if (hostileCallsByMap.TryGetValue(map.uniqueID, out var dict) &&
					dict.TryGetValue(hostileId, out var existing) &&
					existing != null)
				{
					existing.lastKnownCell = lastKnownCell;
					existing.hasLOS = true;
					existing.investigateMode = false;
					existing.lastUpdateTick = now;
				}
				return;
			}

			// Edge: transitioned from no-LOS to has-LOS
			hostileSeenLostByCallerAndHostile[edgeKey] = cur;

			var call = GetOrCreateHostileCall(map, caller, hostile);
			if (call == null) return;

			call.callerPawnId = callerId;
			call.hostileThingId = hostileId;
			call.lastKnownCell = lastKnownCell;
			call.hasLOS = true;
			call.investigateMode = false;
			call.lastUpdateTick = now;

			// Ensure recipients respond immediately as “high alert / investigate” based on existing behavior.
			// Your repo already has AwarenessCache high-alert gating; use the same pattern.
			PublishHostileCallToRecipients(map, lastKnownCell, hostile);
		}

		public static void NotifyPawnLostSightOfHostile(Pawn caller, Pawn hostile, IntVec3 lastKnownCell)
		{
			if (caller == null || hostile == null) return;
			if (caller.Dead) return;
			if (hostile.Dead) return;
			if (caller.Map == null || hostile.Map == null) return;
			if (caller.Map != hostile.Map) return;

			Map map = caller.Map;
			int now = Find.TickManager.TicksGame;

			PruneHostileCallsIfNeeded(map);

			int callerId = caller.thingIDNumber;
			int hostileId = hostile.thingIDNumber;
			int edgeKey = HostileEdgeKey(callerId, hostileId);

			byte prev = 0;
			hostileSeenLostByCallerAndHostile.TryGetValue(edgeKey, out prev);

			// Edge into lost-sight state: bit0 should become 0, and we should avoid repeated publishes.
			// If we already published lost-sight (bit1 might be used, but we reuse bit0 here).
			if ((prev & 1) == 0)
			{
				// Already in lost-sight state for this caller/hostile: refresh TTL + location.
				if (hostileCallsByMap.TryGetValue(map.uniqueID, out var dict) &&
					dict.TryGetValue(hostileId, out var existing) &&
					existing != null)
				{
					existing.lastKnownCell = lastKnownCell;
					existing.hasLOS = false;
					existing.investigateMode = true;
					existing.lastUpdateTick = now;
				}
				return;
			}

			byte cur = (byte)(prev & 0xFE); // clear bit0 => lost sight
			hostileSeenLostByCallerAndHostile[edgeKey] = cur;

			var call = GetOrCreateHostileCall(map, caller, hostile);
			if (call == null) return;

			call.lastKnownCell = lastKnownCell;
			call.hasLOS = false;
			call.investigateMode = true;
			call.lastUpdateTick = now;

			PublishHostileCallToRecipients(map, lastKnownCell, hostile);
		}

		public static void NotifyPawnHostileEliminated(Pawn caller, Pawn hostile)
		{
			if (caller == null || hostile == null) return;
			if (caller.Map == null || hostile.Map == null) return;
			if (caller.Map != hostile.Map) return;

			Map map = caller.Map;
			int now = Find.TickManager.TicksGame;

			int callerId = caller.thingIDNumber;
			int hostileId = hostile.thingIDNumber;
			int edgeKey = HostileEdgeKey(callerId, hostileId);

			byte prev = 0;
			hostileSeenLostByCallerAndHostile.TryGetValue(edgeKey, out prev);

			// Only publish cancel on edge from not-yet-eliminated.
			if ((prev & 0x02) == 0x02)
				return;

			hostileSeenLostByCallerAndHostile[edgeKey] = (byte)(prev | 0x02);

			// Invalidate hostile call immediately
			if (hostileCallsByMap.TryGetValue(map.uniqueID, out var dict))
			{
				dict.Remove(hostileId);
			}

			// Tell recipients to stop investigating this specific hostile last-known.
			var cache = GetOrCreate(map);
			if (cache != null)
			{
				PruneRecipients(cache, map);
				for (int i = cache.recipients.Count - 1; i >= 0; i--)
				{
					Pawn recipient = cache.recipients[i];
					if (recipient == null) continue;
					if (recipient.Dead) continue;
					if (recipient.Map != map) continue;

					ArgrillianThreatState.AwarenessCache.ClearIfHostileMatches(recipient, hostileId);
				}
			}
		}

		// ---- temp helper buffer to avoid changing too much structure yet ----
		private static readonly List<Pawn> recipientsBufferClearOrder = new List<Pawn>(64);

		// ----------------------------
		// Hostile "call" lifecycle (event/edge based)
		// ----------------------------

		// Edge coalescing to avoid per-tick spam. Keyed by (callerPawnId, hostileThingId).
		private static readonly Dictionary<int, int> hostileLastPublishTick = new Dictionary<int, int>();
		private const int HostileEdgePublishCoalesceTicks = 15;

		private static bool CanPublishHostileEdge(int edgeKey)
		{
			int now = Find.TickManager.TicksGame;

			if (hostileLastPublishTick.TryGetValue(edgeKey, out int lastTick))
			{
				if (now - lastTick <= HostileEdgePublishCoalesceTicks)
					return false;
			}

			hostileLastPublishTick[edgeKey] = now;
			return true;
		}

		private static void BroadcastSharedInvestigate(
			Pawn source,
			IntVec3 lastKnownCell,
			int hostileId,
			float allyRadius,
			bool squadOnly)
		{
			if (source == null) return;
			if (source.Map == null) return;

			MapCache cache = GetOrCreate(source.Map);
			if (cache == null) return;

			PruneRecipients(cache, source.Map);

			for (int i = 0; i < cache.recipients.Count; i++)
			{
				Pawn p = cache.recipients[i];
				if (p == null) continue;
				if (p.Dead || !p.Spawned) continue;
				if (p.Map != source.Map) continue;
				if (p == source) continue;

				if (p.Faction != source.Faction) continue;

				if (squadOnly)
				{
					CompArgrillianThreatSettings ts = p.GetComp<CompArgrillianThreatSettings>();
					if (ts == null || ts.squadMode != true) continue;
				}

				if (allyRadius < 0f) allyRadius = 0f;
				if (p.Position.DistanceTo(source.Position) > allyRadius) continue;

				// Don’t downgrade high-alert combatants; don’t spam already-investigating pawns.
				if (ArgrillianThreatState.AwarenessCache.IsHighAlert(p) ||
					ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(p))
					continue;

				ArgrillianThreatState.AwarenessCache.MarkShared(p, lastKnownCell, hostileId);
			}
		}
	}

	public readonly struct ArgrillianThreatHostileAcquireContext
	{
		public Pawn Pawn { get; init; }
		public Map Map { get; init; }
		public float ScanRange { get; init; }
		public float AssistAllyScanRange { get; init; }
		public bool SquadMode { get; init; }
		public bool GuardFellowPawns { get; init; }
	}

	public readonly struct ThreatContext
	{
		public Map Map { get; init; }
		public CompArgrillianThreatSettings Settings { get; init; }
		public float PawnHP { get; init; }
		public bool PursueAdvance { get; init; }
		public bool GuardFellowPawns { get; init; }
		public bool SquadMode { get; init; }
	}

	public readonly struct HostileContext
	{
		public Pawn Hostile { get; init; }
		public Verb AttackVerb { get; init; }
		public bool IsRanged { get; init; }
	}

	// Alert: This class is marked for removal.
	public static class ArgrillianThreatTargeting
	{
		/// <summary>
		/// Acquire a hostile target for a pawn to engage.
		/// Key behavior change vs your current approach:
		/// - Never "soft-block" initiating fight behavior just because the hostile is not in weapon range.
		/// - Weapon-range/imminent-threat gating must be handled when choosing the ATTACK job,
		///   not when acquiring/locking the hostile.
		/// </summary>
		public static bool TryAcquireHostileFromExistingSystems(
	Pawn pawn,
	ThreatContext tctx,
	ArgrillianThreatHostileAcquireContext acquireCtx,
	out HostileContext hctx)
		{
			hctx = default;

			if (pawn == null || pawn.Dead || pawn.Map == null)
				return false;

			var map = acquireCtx.Map;
			float scanRange = acquireCtx.ScanRange;

			if (map == null || pawn.Map != map)
				return false;

			// Read toggles once (from threat settings)
			bool huntHumans = tctx.Settings != null && tctx.Settings.huntHumans;

			// If we already have a locked hostile, prioritize continuing it.
			Pawn hostileLockedContext = ArgrillianThreatState.CombatLock.TryGetLockedHostile(pawn);
			bool hasCombatContext = hostileLockedContext != null && !hostileLockedContext.Dead;

			Pawn hostile = null;

			// Guard/assist logic:
			// Your previous code only checked "guard fellow pawns under threat" when there's no combat context.
			// We keep combat-context priority, but otherwise allow allies under threat to recruit a hostile attacker.
			if (!hasCombatContext && acquireCtx.GuardFellowPawns)
			{
				Pawn allyUnderThreat = FindFriendlyUnderThreat(pawn, map, acquireCtx.AssistAllyScanRange, acquireCtx.SquadMode);
				if (allyUnderThreat != null)
				{
					// Attack whoever is attacking that ally.
					hostile = FindHostileAttackerTo(allyUnderThreat, map, 60f);
				}
			}

			// If we didn't get a hostile from assist logic, try normal acquisition.
			if (hostile == null)
				hostile = FindHostileWithinRange(pawn, map, scanRange, huntHumans);

			// If still none, fall back to lock.
			if (hostile == null)
				hostile = hostileLockedContext;

			if (hostile == null || hostile.Dead)
				return false;

			if (!hostile.Spawned || hostile.Map == null || hostile.Map != map)
				return false;

			if (!hostile.Position.InBounds(map))
				return false;

			if (!pawn.Spawned)
				return false;

			// Mark seen/telemetry for your systems.
			ArgrillianThreatState.CombatLock.MarkSeen(pawn, hostile);

			Verb v = pawn.TryGetAttackVerb(hostile);
			if (v == null)
				return false;

			hctx = new HostileContext
			{
				Hostile = hostile,
				AttackVerb = v,
				IsRanged = v is Verb_Shoot
			};

			return true;
		}

		private static Pawn FindHostileWithinRange(Pawn pawn, Map map, float range, bool huntHumans)
		{
			if (pawn == null || map == null)
				return null;

			CompArgrillianThreatSettings settings = pawn.GetComp<CompArgrillianThreatSettings>();
			bool allowFinishOff = settings != null && settings.finishOff;

			Pawn best = null;
			float bestScore = float.NegativeInfinity;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, range, true))
			{
				if (!c.InBounds(map) || c.Fogged(map))
					continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn other)
						continue;

					if (other.Dead)
						continue;

					// NEW: acquisition-level downed filter to prevent fight-mode anchors/hovering.
					if (other.Downed && !allowFinishOff)
						continue;

					if (!other.HostileTo(pawn))
						continue;

					if (huntHumans && other.RaceProps != null && !other.RaceProps.Humanlike)
						continue;

					bool pawnSeen = GenSight.LineOfSight(other.Position, pawn.Position, map);
					float d = other.Position.DistanceTo(pawn.Position);

					float score = (pawnSeen ? 150f : 50f) - d;

					if (score > bestScore)
					{
						bestScore = score;
						best = other;
					}
				}
			}

			return best;
		}

		private static Pawn FindFriendlyUnderThreat(Pawn pawn, Map map, float range, bool squadMode)
		{
			if (pawn == null || map == null)
				return null;

			Pawn best = null;
			float bestScore = float.NegativeInfinity;

			// If squadMode, bias toward allies who are both low HP and close enough to matter.
			// (Still: do not prevent acquiring hostiles; just choose which ally to respond to.)
			const float squadLowHpThreshold = 0.11f;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, range, true))
			{
				if (!c.InBounds(map) || c.Fogged(map))
					continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn friend) continue;
					if (friend == pawn || friend.Dead) continue;
					if (friend.Faction != pawn.Faction) continue;

					if (squadMode)
					{
						float hpPct = friend.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
						if (hpPct <= squadLowHpThreshold)
						{
							float score = 250f - pawn.Position.DistanceTo(friend.Position) * 2.2f;
							if (score > bestScore)
							{
								bestScore = score;
								best = friend;
							}
							continue;
						}
					}

					// Otherwise: respond if the friend is currently being attacked.
					// We detect by searching for hostile attackers around the friend.
					Pawn attacker = FindHostileAttackerTo(friend, map, 30f);
					if (attacker == null)
						continue;

					bool friendSeen = GenSight.LineOfSight(attacker.Position, friend.Position, map);
					float d = pawn.Position.DistanceTo(friend.Position);

					float threatScore = (friendSeen ? 80f : 30f) - d;
					if (threatScore > bestScore)
					{
						bestScore = threatScore;
						best = friend;
					}
				}
			}

			return best;
		}

		private static Pawn FindHostileAttackerTo(Pawn victim, Map map, float range)
		{
			if (victim == null || victim.Dead || map == null)
				return null;

			// Get finishOff from the *victim*'s settings proxy (victim AI uses same comp type).
			// This method is used as “who is attacking victim”, and should also ignore downed attackers
			// unless finishOff is enabled.
			CompArgrillianThreatSettings settings = victim.GetComp<CompArgrillianThreatSettings>();
			bool allowFinishOff = settings != null && settings.finishOff;

			Pawn best = null;
			float bestScore = float.NegativeInfinity;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(victim.Position, range, true))
			{
				if (!c.InBounds(map) || c.Fogged(map))
					continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn other)
						continue;

					if (other.Dead)
						continue;

					// NEW: acquisition-level downed filter.
					if (other.Downed && !allowFinishOff)
						continue;

					if (!other.HostileTo(victim))
						continue;

					bool victimSeen = GenSight.LineOfSight(other.Position, victim.Position, map);
					float d = other.Position.DistanceTo(victim.Position);

					float score = (victimSeen ? 120f : 40f) - d;
					if (score > bestScore)
					{
						bestScore = score;
						best = other;
					}
				}
			}

			return best;
		}
	}

	public static class ArgrillianThreatHelpers
	{
		public static bool TryBuildContext(Pawn pawn, out ThreatContext ctx)
		{
			ctx = default;

			if (pawn == null || pawn.Dead || pawn.Map == null) return false;
			if (pawn.Downed) return false;

			var s = pawn.GetComp<CompArgrillianThreatSettings>();
			ctx = new ThreatContext
			{
				Map = pawn.Map,
				Settings = s,
				PawnHP = pawn.health?.summaryHealth?.SummaryHealthPercent ?? 1f,
				PursueAdvance = s?.pursueAdvance ?? true,
				GuardFellowPawns = s?.guardFellowPawns ?? true,
				SquadMode = s?.squadMode ?? false
			};

			return true;
		}

		public static bool WantsPatientRetreat(float pawnHP, float minHPToTreatAsPatient)
			=> pawnHP <= minHPToTreatAsPatient;
	}

	public static class ArgrillianThreatMode
	{
		// mode: 0=fight, 1=patient-retreat
		public static byte UpdateMode(
			Pawn pawn,
			float pawnHP,
			float patientRetreatMinHPPercentToTreatAsPatient,
			float patientRetreatMinHPPercentToLockIn,
			int patientRetreatModeHysteresisTicks,
			int PatientFightLockoutAfterRetreatTicks,
			out bool wantsPatientRetreat)
		{
			byte currentMode = ArgrillianThreatState.ModeTickCache.GetMode(pawn);

			wantsPatientRetreat =
				ArgrillianThreatHelpers.WantsPatientRetreat(pawnHP, patientRetreatMinHPPercentToTreatAsPatient);

			// lock-in if already retreating and still fairly injured
			if (wantsPatientRetreat && currentMode == 1 && pawnHP > patientRetreatMinHPPercentToLockIn)
			{
				wantsPatientRetreat = true;
			}
			// if not retreating but we recently were, apply hysteresis / reduce flip-flop
			else if (!wantsPatientRetreat && currentMode == 1)
			{
				if (!ArgrillianThreatState.ModeTickCache.CanSwitchMode(pawn, patientRetreatModeHysteresisTicks))
				{
					wantsPatientRetreat = true;
				}
			}

			// mark mode continuously so Execution can rely on it
			if (wantsPatientRetreat)
			{
				ArgrillianThreatState.ModeTickCache.MarkMode(pawn, 1);
				return 1;
			}

			ArgrillianThreatState.ModeTickCache.MarkMode(pawn, 0);
			return 0;
		}
	}

	public static class ArgrillianThreatExecution
	{
		// -------- NEW: Injured stop-attacking gate --------
		private static bool IsInjuredPatientOrInjuredMedicStopAttacking(
			Pawn pawn,
			float retreatMinHealthPercent)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return false;

			float hp = pawn.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
			return hp <= retreatMinHealthPercent;
		}

		private static bool IsArgrillianMedicPawn(Pawn pawn)
		{
			if (pawn == null || pawn.Dead) return false;
			var comp = pawn.GetComp<CompArgrillianMedicSettings>();
			return comp != null && comp.isMedic;
		}

		private static bool IsArgrillianCombatMedicPawn(Pawn pawn)
		{
			if (pawn == null || pawn.Dead) return false;
			var comp = pawn.GetComp<CompArgrillianMedicSettings>();
			return comp != null && comp.combatMedic;
		}

		// -------- NEW: “imminent threat” check --------
		private static bool IsImminentThreatToTarget(
			Pawn hostile,
			Pawn target,
			float scanRange)
		{
			if (hostile == null || target == null) return false;
			if (hostile.Dead || !hostile.Spawned) return false;
			if (target.Dead || !target.Spawned) return false;
			if (hostile.Map == null || target.Map == null) return false;
			if (hostile.Map != target.Map) return false;

			// If target is not reachable/visible etc., keep logic simple:
			// - For shooting, CanHitTarget already accounts for line/cover/range.
			// - Still, requiring some LOS makes it feel “imminent”.
			Map map = target.Map;

			// Cheap distance gate (prevents expensive checks from far away).
			float dist = hostile.Position.DistanceTo(target.Position);
			if (dist > scanRange) return false;

			// Make sure we can even consider an attack.
			Verb attackVerb = hostile.TryGetAttackVerb(target);
			if (attackVerb == null) return false;

			// If LOS is not present, require that CanHitTarget still succeeds (some mods/verbs may allow otherwise).
			// This is the key “imminent threat” definition you can tune.
			if (!GenSight.LineOfSight(hostile.Position, target.Position, map))
			{
				// Only treat as imminent if the verb can still hit (defensive vs LOS-required verbs).
				if (!attackVerb.CanHitTarget(target))
					return false;

				// Otherwise, verb says it can hit anyway.
			}

			// Core imminence: hostile can currently hit the target.
			return attackVerb.CanHitTarget(target);
		}

		// -------- NEW: “Is this pawn currently being tended by one of my medics?” --------
		private static bool IsBeingTendedByArgrillianMedic(Pawn patient)
		{
			if (patient == null || patient.Map == null || patient.Dead || !patient.Spawned)
				return false;

			return ArgrillianAlertSystem.IsPatientHeldByAnyMedic(patient.Map, patient.thingIDNumber);
		}

		// -------- Patient retreat --------
		public static Job ExecutePatientRetreat(
			Pawn pawn,
			ThreatContext tctx,
			HostileContext hctx,
			float desiredCombatDistanceNow,
			bool pursueAdvance,
			bool skipDecisionTick,
			Pawn nearestMedic,
			float patientRetreatSafeDistanceFromHostile,
			float patientRetreatSearchRadius,
			float patientRetreatPreferMedicRadius,
			float patientRetreatMinHPPercentToTreatAsPatient,
			float patientRetreatMinHPPercentToLockIn,
			int patientRetreatModeHysteresisTicks,
			int PatientFightLockoutAfterRetreatTicks,
			float retreatMinHealthPercent,
			float losBreakBonus,
			float scanRange)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return null;
			if (pawn.Downed) return null;

			// Patient-only: if an Argrillian medic is actively tending/rescuing this patient,
			// patient must hold position (prevents retreat/cover job lock-in).
			if (IsBeingTendedByArgrillianMedic(pawn))
			{
				if (pawn.CurJob != null)
				{
					var def = pawn.CurJob.def;
					if (def == JobDefOf.TendPatient || def == JobDefOf.Rescue)
						return pawn.CurJob;

					// If forced into movement during tend, stop it.
					if (def == JobDefOf.Goto)
						return pawn.CurJob;
				}

				ArgrillianThreatState.CombatCommit.Clear(pawn);
				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			var map = pawn.Map;

			// Hysteresis: if we’re recently in fight mode, don’t instantly retreat.
			byte modeNow = ArgrillianThreatState.ModeTickCache.GetMode(pawn);
			if (modeNow == 0 && !ArgrillianThreatState.ModeTickCache.CanSwitchMode(pawn, patientRetreatModeHysteresisTicks))
				return null;

			// Locked hostile support
			Pawn hostileForPatient = hctx.Hostile;
			if (hostileForPatient == null || hostileForPatient.Dead || !hostileForPatient.Spawned || hostileForPatient.Map != map)
				hostileForPatient = ArgrillianThreatState.CombatLock.TryGetLockedHostile(pawn);

			if (hostileForPatient == null || hostileForPatient.Dead || !hostileForPatient.Spawned || hostileForPatient.Map != map)
				return null;

			// Mark mode/lock when we’re actually retreating (prevents jitter)
			ArgrillianThreatState.ModeTickCache.MarkMode(pawn, 1);
			ArgrillianThreatState.CombatLock.MarkSeen(pawn, hostileForPatient);
			ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);

			// --- Lock-in injured: prefer break-LOS tend spot first ---
			float pawnHP = pawn.health?.summaryHealth?.SummaryHealthPercent ?? 1f;

			Pawn med = nearestMedic;
			if (med == null)
			{
				med = FindNearestMedic(pawn);
			}

			float safeDistance = patientRetreatSafeDistanceFromHostile;

			// If low HP, prefer an LOS-breaking cover cell BEFORE anything else.
			if (pawnHP <= patientRetreatMinHPPercentToLockIn)
			{
				if (TryPickCoverCell(pawn, hostileForPatient, desiredCombatDistanceNow, losBreakBonus, out IntVec3 lockCoverCell))
				{
					var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, lockCoverCell);
					if (keep != null) return keep;

					ArgrillianThreatState.CombatCommit.Clear(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, lockCoverCell);
				}

				// Fallback: keep your original "stop moving so medics can tend" intent.
				if (med != null && !med.Dead && med.Spawned && med.Map == map)
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
				// If no medic exists, still retreat (safe cell picking below).
			}

			// --- Not lock-in yet: prefer break-LOS cover first, then safe retreat ---
			// This aligns retreat movement with your fight-mode LOS-breaking geometry.
			if (pawnHP > patientRetreatMinHPPercentToLockIn)
			{
				if (TryPickCoverCell(pawn, hostileForPatient, desiredCombatDistanceNow, losBreakBonus, out IntVec3 preLockCoverCell))
				{
					var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, preLockCoverCell);
					if (keep != null) return keep;

					ArgrillianThreatState.CombatCommit.Clear(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, preLockCoverCell);
				}
			}

			if (TryPickPatientSafeRetreatCell(
					pawn,
					hostileForPatient,
					med,
					map,
					safeDistance,
					patientRetreatSearchRadius,
					out IntVec3 safeCell))
			{
				var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, safeCell);
				if (keep != null) return keep;

				ArgrillianThreatState.CombatCommit.Clear(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, safeCell);
			}

			return null;
		}

		// -------- Fight mode (UPDATED gate + melee kite abort) --------
		public static Job ExecuteFightMode(
			Pawn pawn,
			ThreatContext tctx,
			HostileContext hctx,
			float desiredCombatDistanceNow,
			bool pursueAdvance,
			bool guardFellowPawns,
			bool squadMode,
			bool skipAggressiveStart,
			float losBreakBonus,
			float retreatMinHealthPercent,
			float minStepCooldownTicks,
			float scanRange,
			float approachSlackMultiplier,
			float meleeChaseMultiplier,
			float meleeProtectRestrictMultiplier,
			float meleePopOutDistanceFactor,
			float rangedPursuitCloseFactor,
			float rangedPursuitMinApproachMultiplier,
			float rangedFiringBandFactor,
			int PatientFightLockoutAfterRetreatTicks)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return null;
			if (pawn.Downed) return null;

			var map = pawn.Map;

			// Locked hostile support
			Pawn hostile = hctx.Hostile;
			if (hostile == null || hostile.Dead || !hostile.Spawned || hostile.Map != map)
				hostile = ArgrillianThreatState.CombatLock.TryGetLockedHostile(pawn);

			if (hostile == null || hostile.Dead || !hostile.Spawned || hostile.Map != map)
				return null;

			Verb v = hctx.AttackVerb;
			if (v == null) return null;

			bool isRanged = hctx.IsRanged;

			// NEW (patient-only): if a medic is actively tending this pawn, patient must hold position
			if (!IsArgrillianMedicPawn(pawn) && IsBeingTendedByArgrillianMedic(pawn))
			{
				// If already on a tend/rescue, let it continue.
				if (pawn.CurJob != null)
				{
					var def = pawn.CurJob.def;
					if (def == JobDefOf.TendPatient || def == JobDefOf.Rescue)
						return pawn.CurJob;

					// If somehow forced into movement during tend, stop it.
					if (def == JobDefOf.Goto)
						return pawn.CurJob;
				}

				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			// EXISTING lockout short-circuit (keep your current behavior)
			bool retreatLockout = ArgrillianThreatState.FightLockoutAfterRetreat
				.RecentlyEndedRetreat(pawn, ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks);

			if (retreatLockout)
			{
				if (pawn.CurJob != null)
				{
					var def = pawn.CurJob.def;
					if (def == JobDefOf.TendPatient || def == JobDefOf.Rescue)
						return pawn.CurJob;
					if (def == JobDefOf.Goto)
						return pawn.CurJob;
				}

				skipAggressiveStart = true;

				if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
				{
					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
					if (keep != null) return keep;

					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
				}

				var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
				if (keepHold != null) return keepHold;

				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			// NEW CHANGE: injuredGate tuning for medics
			// - Patients: use retreatMinHealthPercent as-is
			// - Medics/combat medics: require them to be more injured before they stop attacking.
			// This prevents melee combat medics from immediately falling into "cover/hold/tend" behavior.
			float medicInjuredMultiplier = IsArgrillianCombatMedicPawn(pawn) ? 0.55f : 0.7f;
			float effectiveRetreatMinHP = retreatMinHealthPercent;

			if (IsArgrillianMedicPawn(pawn))
				effectiveRetreatMinHP = retreatMinHealthPercent * medicInjuredMultiplier;

			bool injuredGate = IsInjuredPatientOrInjuredMedicStopAttacking(pawn, effectiveRetreatMinHP);

			if (injuredGate)
			{
				bool imminent = IsImminentThreatToTarget(hostile, pawn, scanRange);

				if (!imminent)
				{
					skipAggressiveStart = true;

					if (pawn.CurJob != null)
					{
						var def = pawn.CurJob.def;
						if (def == JobDefOf.TendPatient || def == JobDefOf.Rescue)
							return pawn.CurJob;
						if (def == JobDefOf.Goto)
							return pawn.CurJob;
					}

					if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
					{
						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
						if (keep != null) return keep;

						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
					}

					var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
					if (keepHold != null) return keepHold;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
				}
			}

			// NORMAL FIGHT MODE
			ArgrillianThreatState.ModeTickCache.MarkMode(pawn, 0);
			ArgrillianThreatState.CombatLock.MarkSeen(pawn, hostile);
			ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
			
			// NEW: downed-hostile gate must apply even when skipAggressiveStart == true.
			// Otherwise melee/positioning can still orbit the downed anchor.
			if (!IsArgrillianMedicPawn(pawn) && hostile != null && hostile.Downed && !isRanged)
			{
				CompArgrillianThreatSettings comp = pawn.GetComp<CompArgrillianThreatSettings>();
				bool finishOff = comp != null && comp.finishOff;

				if (!finishOff)
				{
					return null;
				}
			}

			if (ArgrillianThreatState.CombatCommit.RecentlyCommitted(pawn) &&
				ArgrillianThreatState.CombatCommit.CommitMatchesHostile(pawn, hostile))
			{
				// keep ongoing attack job logic below
			}

			if (!skipAggressiveStart)
			{
				// Keep ongoing attack job, unless FinishOff is OFF and the hostile is downed.
				if (pawn.CurJob != null &&
					(pawn.CurJob.def == JobDefOf.AttackMelee || pawn.CurJob.def == JobDefOf.AttackStatic))
				{
					if (hostile != null && hostile.Downed)
					{
						CompArgrillianThreatSettings comp = pawn.GetComp<CompArgrillianThreatSettings>();
						bool finishOff = comp != null && comp.finishOff;

						if (!finishOff)
						{
							// Do not derail combat-medic tending behavior.
							CompArgrillianMedicSettings medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
							bool combatMedic = medicComp != null && medicComp.combatMedic;

							if (!combatMedic)
							{
								pawn.jobs?.StopAll(true); 
								pawn.pather?.StopDead();
								return null;
							}
						}
					}

					return pawn.CurJob;
				}

				Job attackJob = TryMakeAttackJobIfCanHitNow(
					pawn,
					hostile,
					pursueAdvance,
					desiredCombatDistanceNow,
					isRanged);

				if (attackJob != null)
				{
					ArgrillianThreatState.CombatCommit.Mark(pawn, hostile);
					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return attackJob;
				}

				// HARD GATE FIX:
				// FinishOff OFF => ignore downed hostiles (stop orbiting/moving around them),
				// but DO NOT abort combat-medic logic. Medics must stay in their
				// patient-tending flow for downed colonists.
				if (hostile != null && hostile.Downed)
				{
					CompArgrillianThreatSettings comp = pawn.GetComp<CompArgrillianThreatSettings>();
					bool finishOff = comp != null && comp.finishOff;

					if (!finishOff)
					{
						CompArgrillianMedicSettings medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
						bool combatMedic = medicComp != null && medicComp.combatMedic;

						if (!combatMedic)
							return null;
					}
				}
			}

			// -------- Squad handling (unchanged from your provided block) --------
			if (squadMode)
			{
				Pawn anchor = SquadAnchorFor(pawn, hostile);
				if (anchor != null && anchor != pawn && !anchor.Dead && anchor.Map == map)
				{
					Vector3 toHostile = hostile.Position.ToVector3Shifted() - anchor.Position.ToVector3Shifted();
					if (toHostile.sqrMagnitude < 0.0001f)
						toHostile = hostile.Position.ToVector3Shifted() - pawn.Position.ToVector3Shifted();

					toHostile.y = 0f;

					Vector3 forward = toHostile.normalized;
					Vector3 right = Vector3.Cross(Vector3.up, forward).normalized;

					int slot = SquadSlotIndex(pawn, anchor);
					float ringRadius = Mathf.Max(2.5f, desiredCombatDistanceNow * (pursueAdvance ? 1.6f : 1.15f));
					float strafeStep = pursueAdvance ? 1.8f : 1.25f;

					int wave = slot / 2 + 1;
					int side = (slot % 2 == 0) ? 1 : -1;

					Vector3 desiredPoint = anchor.Position.ToVector3Shifted()
						+ forward * (desiredCombatDistanceNow * (pursueAdvance ? 0.65f : 0.35f))
						+ right * (side * wave * strafeStep);

					IntVec3 desiredCell = desiredPoint.ToIntVec3();

					IntVec3 finalCell = pawn.Position;
					if (TryPickNearestWalkableStandable(pawn, desiredCell, 6, out IntVec3 picked))
						finalCell = picked;

					if (isRanged)
					{
						if (pursueAdvance)
						{
							if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
							{
								if (finalCell.DistanceTo(coverCell) <= 8f)
								{
									var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
									if (keep != null) return keep;

									ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
									return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
								}
							}

							ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
							if (TryPickApproachCell(pawn, hostile, desiredCombatDistanceNow, out IntVec3 approachCell))
							{
								var keep2 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, approachCell);
								if (keep2 != null) return keep2;

								return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, approachCell);
							}
						}
						else
						{
							if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
							{
								var keepCover = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
								if (keepCover != null) return keepCover;

								ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
								return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
							}
						}

						var keepFinalRanged = ArgrillianGotoHelper.KeepIfSameGoto(pawn, finalCell);
						if (keepFinalRanged != null) return keepFinalRanged;

						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, finalCell);
					}
					else
					{
						// melee squad movement uses your original structure
						if (!pursueAdvance)
						{
							if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
							{
								var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
								if (keep != null) return keep;

								ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
								return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
							}

							var keep2Protect = ArgrillianGotoHelper.KeepIfSameGoto(pawn, finalCell);
							if (keep2Protect != null) return keep2Protect;

							ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
							return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, finalCell);
						}
						else
						{
							if (pursueAdvance)
							{
								if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell3))
								{
									if (finalCell.DistanceTo(coverCell3) <= 6f)
									{
										var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell3);
										if (keep != null) return keep;

										ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
										return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell3);
									}
								}

								ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
								var keep3 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, finalCell);
								if (keep3 != null) return keep3;

								return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, finalCell);
							}
							else
							{
								if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
								{
									var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
									if (keep != null) return keep;

									ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
									return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
								}

								var keep2Protect = ArgrillianGotoHelper.KeepIfSameGoto(pawn, finalCell);
								if (keep2Protect != null) return keep2Protect;

								ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
								return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, finalCell);
							}
						}
					}
				}
			}

			// -------- movement band decisions (unchanged) --------
			if (isRanged)
			{
				// (your entire ranged block unchanged)
				if (!pursueAdvance)
				{
					if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCellNoPursue))
					{
						var keepCover = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCellNoPursue);
						if (keepCover != null) return keepCover;

						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCellNoPursue);
					}

					var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
					if (keepHold != null) return keepHold;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					IntVec3 tiny = pawn.Position + ArgrillianThreatGeometry.GetAdj8()[Rand.Range(0, ArgrillianThreatGeometry.GetAdj8().Length)];
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, tiny);
				}

				if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
				{
					if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != coverCell)
					{
						if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, coverCell))
							return pawn.CurJob;
					}

					var keepCover = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
					if (keepCover != null) return keepCover;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
				}

				float targetDistForApproach = desiredCombatDistanceNow * rangedPursuitCloseFactor;
				targetDistForApproach = Mathf.Max(desiredCombatDistanceNow * rangedPursuitMinApproachMultiplier, targetDistForApproach);

				if (TryPickApproachCell(pawn, hostile, targetDistForApproach, out IntVec3 approachCell))
				{
					if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != approachCell)
					{
						if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, approachCell))
							return pawn.CurJob;
					}

					var keepApproach = ArgrillianGotoHelper.KeepIfSameGoto(pawn, approachCell);
					if (keepApproach != null) return keepApproach;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, approachCell);
				}

				if (TryPickApproachCell(pawn, hostile, desiredCombatDistanceNow * 1.05f, out IntVec3 approachCell2))
				{
					if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != approachCell2)
					{
						if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, approachCell2))
							return pawn.CurJob;
					}

					var keepApproach2 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, approachCell2);
					if (keepApproach2 != null) return keepApproach2;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, approachCell2);
				}

				var keepRangedPos = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
				if (keepRangedPos != null) return keepRangedPos;

				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			// melee
			float dist = pawn.Position.DistanceTo(hostile.Position);

			// -------- MELEE KITE ABORT / RE-CHASE RE-ENTRY --------
			// If we're currently trying to melee-attack/chase, but we remain out of melee band for N ticks,
			// clear combat commit and go back to chase/approach movement.
			// This is intentionally suppressed by injuredGate (since early code returned cover/hold when not imminent).
			if (pawn.CurJob != null && (pawn.CurJob.def == JobDefOf.AttackMelee || pawn.CurJob.def == JobDefOf.AttackStatic || pawn.CurJob.def == JobDefOf.Goto))
			{
				// Define “melee band”:
				// - inside if within [desired*0.85, desired*bandMax]
				// - out if too far OR too close (kite tends to increase distance, so the too-far case is most important)
				float bandMin = desiredCombatDistanceNow * 0.85f;
				float bandMax = desiredCombatDistanceNow * (pursueAdvance ? meleeChaseMultiplier : meleeProtectRestrictMultiplier);

				bool currentlyOutOfBand = (dist < bandMin) || (dist > bandMax);

				// N ticks: use minStepCooldownTicks as your tuning proxy (it already exists in your signature).
				// If you want a dedicated config, replace this with a param (e.g. meleeKiteAbortTicks).
				int requiredTicks = Mathf.Max(10, (int)minStepCooldownTicks); // at least 10 ticks

				int currentTick = Find.TickManager.TicksGame;

				if (ArgrillianThreatState.MeleeKiteAbortTickCache.IsOutOfBandLongEnough(pawn, currentlyOutOfBand, requiredTicks, currentTick, out int _))
				{
					// Clear commit/lock so we can re-plan movement and re-enter melee.
					ArgrillianThreatState.CombatCommit.Clear(pawn);

					// Also reset cache so we don’t repeatedly spam this on subsequent ticks.
					ArgrillianThreatState.MeleeKiteAbortTickCache.Reset(pawn);

					// Choose a chase/approach cell (push back toward desired combat distance).
					if (TryPickApproachCell(pawn, hostile, desiredCombatDistanceNow, out IntVec3 chaseCell))
					{
						var keepGoto = ArgrillianGotoHelper.KeepIfSameGoto(pawn, chaseCell);
						if (keepGoto != null) return keepGoto;

						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, chaseCell);
					}

					// Fallback: keep current position (still clears combat commit above)
					var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
					if (keepHold != null) return keepHold;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
				}
				else
				{
					// If we're back in band, reset the timer.
					if (!currentlyOutOfBand)
						ArgrillianThreatState.MeleeKiteAbortTickCache.Reset(pawn);
				}
			}

			if (!pursueAdvance)
			{
				bool closeEnoughToPop = dist <= desiredCombatDistanceNow * meleePopOutDistanceFactor;

				if (!closeEnoughToPop)
				{
					if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
					{
						var keepCover = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
						if (keepCover != null) return keepCover;

						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
					}

					var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
					if (keepHold != null) return keepHold;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
				}
			}

			float desiredBandMult = approachSlackMultiplier;
			desiredBandMult *= pursueAdvance ? meleeChaseMultiplier : meleeProtectRestrictMultiplier;

			if (dist > desiredCombatDistanceNow * desiredBandMult)
			{
				if (!pursueAdvance)
				{
					if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCellNoPursue))
					{
						var keepCover2 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCellNoPursue);
						if (keepCover2 != null) return keepCover2;

						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCellNoPursue);
					}

					var keepHold = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
					if (keepHold != null) return keepHold;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
				}

				if (TryPickApproachCell(pawn, hostile, desiredCombatDistanceNow, out IntVec3 approachCell))
				{
					if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != approachCell)
					{
						if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, approachCell))
							return pawn.CurJob;
					}

					var keepApproach = ArgrillianGotoHelper.KeepIfSameGoto(pawn, approachCell);
					if (keepApproach != null) return keepApproach;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, approachCell);
				}
			}

			if (TryPickCoverCell(pawn, hostile, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell2))
			{
				if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != coverCell2)
				{
					if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, coverCell2))
						return pawn.CurJob;
				}

				var keepCover2Final = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell2);
				if (keepCover2Final != null) return keepCover2Final;

				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell2);
			}

			if (pursueAdvance)
			{
				if (TryPickApproachCell(pawn, hostile, desiredCombatDistanceNow, out IntVec3 approachCell3))
				{
					if (pawn.CurJob?.def == JobDefOf.Goto && pawn.CurJob.targetA.Cell != approachCell3)
					{
						if (!ArgrillianGotoHelper.RepathCooldown.ShouldAllowNewDestination(pawn, approachCell3))
							return pawn.CurJob;
					}

					var keepApproach3 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, approachCell3);
					if (keepApproach3 != null) return keepApproach3;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, approachCell3);
				}
			}

			var keepFinalMelee = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
			if (keepFinalMelee != null) return keepFinalMelee;

			ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
			return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
		}

		// -------- Alerted mode (UPDATED gate) --------
		public static Job ExecuteAlertedMode(
			Pawn pawn,
			Pawn hostileIfAny,
			float desiredCombatDistanceNow,
			float losBreakBonus,
			bool isRanged,
			int minHighAlertTicksToMove,
			float scanRange,
			float retreatMinHealthPercent,
			float lastKnownInvestigateRadius = 12f)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return null;
			if (pawn.Downed) return null;

			// MEDIC-ONLY override (conditioned):
			// If this medic's own job is Tend/Rescue, keep it as long as the target is still in the medic's medical pipeline.
			if (IsArgrillianMedicPawn(pawn) && pawn.CurJob != null)
			{
				JobDef def = pawn.CurJob.def;
				if (def == JobDefOf.TendPatient || def == JobDefOf.Rescue)
				{
					Pawn patient = pawn.CurJob.targetA.Pawn;
					if (patient != null && !patient.Dead && patient.Spawned && patient.Map == pawn.Map)
					{
						// Use your authoritative hold mapping instead of IsBeingTendedByArgrillianMedic(),
						// which may be out of sync with PatientMedicHold.
						Pawn held = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);
						bool patientStillBoundToMedic = held == patient;

						// Rescue is for downed/unable patients.
						if (def == JobDefOf.Rescue)
							patientStillBoundToMedic = patientStillBoundToMedic && patient.Downed;

						if (patientStillBoundToMedic)
							return pawn.CurJob;
					}
				}
			}

			Map map = pawn.Map;

			bool highAlert = ArgrillianThreatState.AwarenessCache.IsHighAlert(pawn);
			bool sharedInvestigate = ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(pawn);
			if (!highAlert && !sharedInvestigate)
			{
				return null;
			}

			// Resolve hostile if given and valid
			if (hostileIfAny != null && !hostileIfAny.Dead && hostileIfAny.Spawned && hostileIfAny.Map == map)
			{
				bool injuredCombatMedicStop = false;

				var medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
				if (medicComp != null && medicComp.isMedic && medicComp.combatMedic)
				{
					injuredCombatMedicStop = IsInjuredPatientOrInjuredMedicStopAttacking(
						pawn,
						retreatMinHealthPercent
					);
				}

				bool hasLOS = GenSight.LineOfSight(pawn.Position, hostileIfAny.Position, map);

				if (!hostileIfAny.Dead && hostileIfAny.Spawned && hostileIfAny.Map == pawn.Map)
				{
					if (hasLOS)
						ArgrillianAlertSystem.NotifyPawnSeesHostile(pawn, hostileIfAny, hostileIfAny.Position);
					else
						ArgrillianAlertSystem.NotifyPawnLostSightOfHostile(pawn, hostileIfAny, hostileIfAny.Position);
				}

				if (hasLOS && !injuredCombatMedicStop)
				{
					// Try to start a real attack job immediately.
					bool pursueAdvance = false; // alerted "first contact" usually handled by movement logic below
					Job attackJob = ArgrillianThreatExecution.TryMakeAttackJobIfCanHitNow(
						pawn,
						hostileIfAny,
						pursueAdvance,
						desiredCombatDistanceNow,
						isRanged
					);

					if (attackJob != null)
						return attackJob;

					// If we can't hit yet, fall through to cover/last-known reposition logic below.
				}

				// injuredCombatMedicStop OR no-LOS:
				// fall through to investigate/reposition logic below (won't initiate attack).
			}

			// Otherwise investigate/reposition
			IntVec3 lastKnown = ArgrillianThreatState.AwarenessCache.GetLastKnownCell(pawn);
			if (!lastKnown.IsValid)
			{
				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			if (TryPickCoverCellFromLastKnown(pawn, lastKnown, desiredCombatDistanceNow, losBreakBonus, out IntVec3 coverCell))
			{
				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, coverCell);
				if (keep != null) return keep;

				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, coverCell);
			}

			if (lastKnownInvestigateRadius > 0f)
			{
				Vector3 dir = (lastKnown.ToVector3Shifted() - pawn.Position.ToVector3Shifted());
				dir.y = 0f;

				if (dir.sqrMagnitude > 0.001f)
				{
					dir.Normalize();

					float dist = Mathf.Clamp(lastKnownInvestigateRadius, 1f, 10f);
					Vector3 stepV = pawn.Position.ToVector3Shifted() + (dir * dist);
					IntVec3 step = stepV.ToIntVec3();

					if (TryPickNearestWalkableStandable(pawn, step, 6, out IntVec3 picked))
					{
						ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
						return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, picked);
					}
				}
			}

			ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
			return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
		}

		private static bool TryPickCoverCellFromLastKnown(
			Pawn pawn,
			IntVec3 lastKnownHostileCell,
			float desiredDistance,
			float losBreakBonus,
			out IntVec3 bestCell)
		{
			bestCell = pawn.Position;
			Map map = pawn.Map;
			if (map == null) return false;

			float scanRadius = 18f;
			float bestScore = float.NegativeInfinity;

			Vector3 pawnPos = pawn.Position.ToVector3Shifted();

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, scanRadius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null && c != pawn.Position) continue;
				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				// Score: prefer cells at ~desiredDistance from last known.
				float dist = c.DistanceTo(lastKnownHostileCell);
				float distScore = -Mathf.Abs(dist - desiredDistance);

				// "Cover-ness": break LOS to last known cell.
				bool breaksLOS = !GenSight.LineOfSight(lastKnownHostileCell, c, map);
				float safetyBonus = breaksLOS ? losBreakBonus * 0.25f : 0f;

				// Also avoid moving directly into LOS (weakly).
				// If we can't break LOS, heavily downscore.
				float losPenalty = breaksLOS ? 0f : -40f;

				float score = distScore + safetyBonus + losPenalty;

				if (score > bestScore + 0.01f)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			return bestCell != pawn.Position;
		}

		private static bool TryPickPatientSafeRetreatCell(
			Pawn patient,
			Pawn hostile,
			Pawn nearestMedic,
			Map map,
			float safeDistance,
			float searchRadius,
			out IntVec3 bestCell)
		{
			bestCell = patient.Position;

			if (patient == null || hostile == null) return false;

			float r = ArgrillianThreatMath.ClampRadialRadius(searchRadius);

			float bestScore = float.NegativeInfinity;

			bool hostileSeesPatientNow = GenSight.LineOfSight(hostile.Position, patient.Position, map);

			foreach (IntVec3 c in GenRadial.RadialCellsAround(patient.Position, r, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null && c != patient.Position) continue;
				if (!patient.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				float dToHostile = c.DistanceTo(hostile.Position);

				float distanceScore = dToHostile >= safeDistance
					? (dToHostile - safeDistance) * 6f
					: -(safeDistance - dToHostile) * 4f;

				bool hostileSeesCandidate = GenSight.LineOfSight(hostile.Position, c, map);
				float losScore = hostileSeesCandidate ? -90f : 90f;

				if (hostileSeesPatientNow && !hostileSeesCandidate) losScore += 25f;

				float medicProgress = 0f;
				if (nearestMedic != null && !nearestMedic.Dead && nearestMedic.Spawned && nearestMedic.Map == map)
				{
					float distToMedic = c.DistanceTo(nearestMedic.Position);
					medicProgress = (patient.Position.DistanceTo(nearestMedic.Position) - distToMedic) * 0.8f;
				}

				float closeness = -c.DistanceTo(patient.Position) * 0.08f;

				float score = distanceScore + losScore + medicProgress + closeness;

				if (score > bestScore + 0.01f)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			return bestCell != patient.Position;
		}

		private static Job TryMakeAttackJobIfCanHitNow(
			Pawn pawn,
			Pawn target,
			bool pursueAdvance,
			float desiredCombatDistance,
			bool isRanged)
		{
			if (pawn == null || target == null || pawn.Dead || target.Dead)
			{
				return null;
			}

			// Finish Off toggle:
			// OFF (default): ignore downed hostiles (current behavior).
			// ON: allow Attack* jobs against downed hostiles.
			bool allowFinishOff = false;
			{
				CompArgrillianThreatSettings s = pawn.GetComp<CompArgrillianThreatSettings>();
				allowFinishOff = s != null && s.finishOff;
			}

			if (target.Downed && !allowFinishOff)
			{
				return null;
			}

			if (!pawn.Spawned || pawn.Map == null)
			{
				return null;
			}

			if (!target.Spawned || target.Map == null || pawn.Map != target.Map)
			{
				return null;
			}

			if (!target.Position.InBounds(pawn.Map))
			{
				return null;
			}

			Verb attackVerb = pawn.TryGetAttackVerb(target);
			if (attackVerb == null)
			{
				return null;
			}

			float dist = pawn.Position.DistanceTo(target.Position);

			if (attackVerb is Verb_Shoot shoot)
			{
				if (pursueAdvance && dist > desiredCombatDistance)
				{
					return null;
				}
				float min = shoot.verbProps.minRange;
				float max = shoot.verbProps.range;

				if (max <= 0.01f)
				{
					return null;
				}

				float allowedDist = max * (pursueAdvance ? 1.02f : 1.05f);

				if (dist > allowedDist)
				{
					return null;
				}

				if (min > 0.01f && dist < min)
				{
					return null;
				}

				if (!attackVerb.CanHitTarget(target))
				{
					return null;
				}

				return JobMaker.MakeJob(JobDefOf.AttackStatic, target);
			}

			if (!pursueAdvance)
			{
				float max = attackVerb.verbProps.range;
				float min = attackVerb.verbProps.minRange;

				if (max > 0.01f)
				{
					float allowedDist = Mathf.Max(min + 0.1f, max);
					if (dist > allowedDist)
					{
						return null;
					}
				}
			}

			if (!attackVerb.CanHitTarget(target))
			{
				return null;
			}

			if (pursueAdvance && dist > desiredCombatDistance)
			{
				return null;
			}

			return JobMaker.MakeJob(JobDefOf.AttackMelee, target);
		}

		private static bool TryPickCoverCell(Pawn pawn, Pawn attacker, float desiredDistance, float losBreakBonus, out IntVec3 bestCell)
		{
			bestCell = pawn.Position;
			Map map = pawn.Map;

			bool avoidBeingInFriendlyLineOfFire = true;
			float friendlyLineToleranceTiles = 2.5f;

			float bestScore = float.NegativeInfinity;

			float scanRadius = 18f;
			float friendScanRadius = 25f;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, scanRadius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null) continue;
				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				if (!GenSight.LineOfSight(c, attacker.Position, map)) continue;

				if (avoidBeingInFriendlyLineOfFire && IsCellInFriendlyLineOfFire(pawn, attacker, c, friendlyLineToleranceTiles, friendScanRadius))
					continue;

				float dist = c.DistanceTo(attacker.Position);
				float distScore = -Mathf.Abs(dist - desiredDistance);

				bool breaksLosToPawn = !GenSight.LineOfSight(c, pawn.Position, map);
				float safetyBonus = breaksLosToPawn ? losBreakBonus * 0.25f : 0f;

				float score = distScore + safetyBonus;

				if (score > bestScore + 0.01f)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			return bestCell != pawn.Position;
		}

		private static bool IsCellInFriendlyLineOfFire(Pawn self, Pawn hostile, IntVec3 candidateCell, float friendlyLineToleranceTiles, float friendScanRadius)
		{
			if (self == null || hostile == null || self.Map == null) return false;

			Map map = self.Map;
			Vector3 candidate = candidateCell.ToVector3Shifted();

			float r = Mathf.Max(1f, friendScanRadius);

			foreach (IntVec3 c in GenRadial.RadialCellsAround(self.Position, r, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				foreach (Thing thing in c.GetThingList(map))
				{
					if (thing is not Pawn friend) continue;
					if (friend == self || friend.Dead) continue;
					if (friend.Faction != self.Faction) continue;

					Verb fv = friend.TryGetAttackVerb(hostile);
					if (fv == null || fv is not Verb_Shoot) continue;

					if (!GenSight.LineOfSight(friend.Position, hostile.Position, map)) continue;

					Vector3 shooter = friend.Position.ToVector3Shifted();
					Vector3 hostilePos = hostile.Position.ToVector3Shifted();

					Vector3 seg = hostilePos - shooter;
					float segLenSq = seg.sqrMagnitude;
					if (segLenSq < 0.001f) continue;

					float projT = Vector3.Dot(candidate - shooter, seg) / segLenSq;
					projT = Mathf.Clamp01(projT);

					Vector3 closest = shooter + seg * projT;
					float distToLine = Vector3.Distance(candidate, closest);

					if (distToLine <= friendlyLineToleranceTiles)
						return true;
				}
			}

			return false;
		}

		public static Pawn FindNearestMedic(Pawn seeker)
		{
			if (seeker == null || seeker.Dead || seeker.Map == null) return null;

			Map map = seeker.Map;

			// PERF: avoid map-wide AllPawnsSpawned scans.
			// Use the lazily-pruned registered combat-medic bucket created at toggle time.
			List<Pawn> combatMedics = CompArgrillianMedicSettings.GetCombatMedics(map);
			if (combatMedics == null || combatMedics.Count == 0) return null;

			Pawn best = null;
			float bestD = float.PositiveInfinity;

			for (int i = 0; i < combatMedics.Count; i++)
			{
				Pawn p = combatMedics[i];
				if (p == null || p.Dead) continue;
				if (!p.Spawned || p.Map != map) continue;
				if (p.Faction != seeker.Faction) continue;
				if (p.health == null) continue;

				// Keep old behavior: skip downed.
				if (p.Downed) continue;

				float d = seeker.Position.DistanceTo(p.Position);
				if (d < bestD)
				{
					bestD = d;
					best = p;
				}
			}

			return best;
		}

		private static Pawn SquadAnchorFor(Pawn pawn, Pawn hostile)
		{
			if (pawn == null || hostile == null || pawn.Map == null) return null;

			float anchorSearchRadius = 40f;

			Pawn best = null;
			int bestId = int.MaxValue;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, anchorSearchRadius, true))
			{
				if (!c.InBounds(pawn.Map) || c.Fogged(pawn.Map)) continue;

				foreach (Thing t in c.GetThingList(pawn.Map))
				{
					if (t is not Pawn p) continue;
					if (p.Dead) continue;
					if (p.Faction != pawn.Faction) continue;

					if (p.GetComp<CompArgrillianThreatSettings>()?.squadMode != true) continue;

					int id = p.thingIDNumber;
					if (id < bestId)
					{
						bestId = id;
						best = p;
					}
				}
			}

			return best ?? pawn;
		}

		private static int SquadSlotIndex(Pawn pawn, Pawn anchor)
		{
			if (pawn == null || anchor == null) return 0;

			int anchorId = anchor.thingIDNumber;
			int myId = pawn.thingIDNumber;

			int delta = myId - anchorId;
			if (delta < 0) delta = -delta;

			return delta % 10;
		}

		private static bool TryPickNearestWalkableStandable(Pawn pawn, IntVec3 desired, int radius, out IntVec3 picked)
		{
			picked = pawn.Position;

			Map map = pawn.Map;
			float best = float.PositiveInfinity;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(desired, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null && c != pawn.Position) continue;

				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				float d = c.DistanceTo(desired);
				if (d < best)
				{
					best = d;
					picked = c;
				}
			}

			return picked != pawn.Position;
		}

		// referenced by ExecuteFightMode (your original)
		private static bool TryPickApproachCell(Pawn pawn, Pawn hostile, float desiredDistance, out IntVec3 approachCell)
		{
			approachCell = pawn.Position;
			float bestScore = float.NegativeInfinity;

			Map map = pawn.Map;
			if (map == null) return false;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, 10f, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null && c != pawn.Position) continue;
				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				float d = c.DistanceTo(hostile.Position);
				float score = -Mathf.Abs(d - desiredDistance);

				if (score > bestScore + 0.01f)
				{
					bestScore = score;
					approachCell = c;
				}
			}

			return approachCell != pawn.Position;
		}
	}

	// -----------------------------
	// Threat settings / toggles
	// -----------------------------
	public class CompProperties_ArgrillianThreatSettings : CompProperties
	{
		public CompProperties_ArgrillianThreatSettings() => compClass = typeof(CompArgrillianThreatSettings);
	}

	public class CompArgrillianThreatSettings : ThingComp
	{
		public bool pursueAdvance = true;
		public bool guardFellowPawns = true;

		public bool squadMode = false;

		public bool finishOff = false;
		public bool huntHumans = false;

		private static bool loggedThreatSettingsGizmosOnce;

		public void ExposeData()
		{
			Scribe_Values.Look(ref pursueAdvance, "pursueAdvance", true);
			Scribe_Values.Look(ref guardFellowPawns, "guardFellowPawns", true);
			Scribe_Values.Look(ref squadMode, "squadMode", false);

			Scribe_Values.Look(ref finishOff, "finishOff", false);
			Scribe_Values.Look(ref huntHumans, "huntHumans", false);
		}

		public override IEnumerable<Gizmo> CompGetGizmosExtra()
		{
			var sw = new System.Diagnostics.Stopwatch();
			sw.Start();

			try
			{
				foreach (Gizmo g in base.CompGetGizmosExtra())
					yield return g;

				yield return ArgrillianGizmoHelpers.Toggle(
					"Pursue/Advance",
					"ON: advance/flank/seek firing positions. OFF: take cover/hold until close enough, then pop out.",
					() => pursueAdvance,
					() => pursueAdvance = !pursueAdvance
				);

				yield return ArgrillianGizmoHelpers.Toggle(
					"Guard fellow pawns",
					"ON: assist allies under attack and help them regorge.",
					() => guardFellowPawns,
					() => guardFellowPawns = !guardFellowPawns
				);

				yield return ArgrillianGizmoHelpers.Toggle(
					"Squad mode",
					"ON: keep a tighter unit and reposition together for advantage.",
					() => squadMode,
					() => squadMode = !squadMode
				);

				yield return ArgrillianGizmoHelpers.Toggle(
					"Finish Off",
					"ON: actively kill downed hostiles. OFF: ignore downed hostiles (current behavior).",
					() => finishOff,
					() => finishOff = !finishOff
				);

				yield return ArgrillianGizmoHelpers.Toggle(
					"Hunt",
					"ON: actively hunt human pawns when selecting hostile targets.",
					() => huntHumans,
					() => huntHumans = !huntHumans
				);
			}
			finally
			{
				sw.Stop();

				const double thresholdMs = 5.0;
				if (!loggedThreatSettingsGizmosOnce && sw.Elapsed.TotalMilliseconds >= thresholdMs)
				{
					loggedThreatSettingsGizmosOnce = true;
					Log.Message($"[ArgrillianThreat] CompGetGizmosExtra(ThreatSettings) slow: {sw.Elapsed.TotalMilliseconds:0.00} ms parent={(parent != null ? parent.ToString() : "null")}");
				}
			}
		}
	}

	// -----------------------------
	// Medic settings / marker + UI
	// -----------------------------
	public class CompProperties_ArgrillianMedicSettings : CompProperties
	{
		public CompProperties_ArgrillianMedicSettings() => compClass = typeof(CompArgrillianMedicSettings);
	}

	public class CompArgrillianMedicSettings : ThingComp
	{
		public bool isMedic = false;
		public bool combatMedic = false;
		public int assignedPawnThingID = -1;

		private static readonly Dictionary<int, List<Pawn>> combatMedicsByMapId = new Dictionary<int, List<Pawn>>();
		private static readonly Dictionary<int, HashSet<int>> combatMedicIdsByMapId = new Dictionary<int, HashSet<int>>();

		private static readonly Dictionary<int, List<Pawn>> medicsByMapId = new Dictionary<int, List<Pawn>>();
		private static readonly Dictionary<int, HashSet<int>> medicIdsByMapId = new Dictionary<int, HashSet<int>>();

		private static bool loggedMedicGizmosOnce;

		private static void EnsureMapBuckets(int mapId)
		{
			if (!combatMedicsByMapId.TryGetValue(mapId, out var _))
				combatMedicsByMapId[mapId] = new List<Pawn>(8);
			if (!combatMedicIdsByMapId.TryGetValue(mapId, out var _))
				combatMedicIdsByMapId[mapId] = new HashSet<int>();

			if (!medicsByMapId.TryGetValue(mapId, out var _))
				medicsByMapId[mapId] = new List<Pawn>(8);
			if (!medicIdsByMapId.TryGetValue(mapId, out var _))
				medicIdsByMapId[mapId] = new HashSet<int>();
		}

		private static void RegisterMedic(Pawn pawn)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return;

			int mapId = pawn.Map.uniqueID;
			EnsureMapBuckets(mapId);

			int medicId = pawn.thingIDNumber;

			// General medic list
			if (medicIdsByMapId[mapId].Add(medicId))
				medicsByMapId[mapId].Add(pawn);

			// Combat medic list
			var comp = pawn.GetComp<CompArgrillianMedicSettings>();
			if (comp != null && comp.combatMedic)
			{
				if (combatMedicIdsByMapId[mapId].Add(medicId))
					combatMedicsByMapId[mapId].Add(pawn);
			}
		}

		private static void UnregisterMedic(Pawn pawn)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return;

			int mapId = pawn.Map.uniqueID;
			EnsureMapBuckets(mapId);

			int medicId = pawn.thingIDNumber;

			medicIdsByMapId[mapId].Remove(medicId);
			combatMedicIdsByMapId[mapId].Remove(medicId);

			// We don't aggressively remove from Lists (cheaper). Lookups will skip stale pawns/IDs.
		}

		internal static List<Pawn> GetCombatMedics(Map map)
		{
			if (map == null) return null;

			int mapId = map.uniqueID;
			if (!combatMedicsByMapId.TryGetValue(mapId, out var list)) return null;
			if (!combatMedicIdsByMapId.TryGetValue(mapId, out var ids)) return null;

			// Prune lazily to keep the list from accumulating dead/stale entries.
			// This avoids AllPawnsSpawned scans.
			for (int i = list.Count - 1; i >= 0; i--)
			{
				var p = list[i];
				if (p == null || p.Dead || p.Map != map)
				{
					list.RemoveAt(i);
					continue;
				}

				int pid = p.thingIDNumber;
				var comp = p.GetComp<CompArgrillianMedicSettings>();
				bool stillCombat = comp != null && comp.isMedic && comp.combatMedic;

				if (!ids.Contains(pid) || !stillCombat)
				{
					list.RemoveAt(i);
				}
			}

			return list;
		}

		public Pawn AssignedPawn
		{
			get => ArgrillianGizmoHelpers.FindAlivePawnByThingID(parent, assignedPawnThingID);
		}

		public override IEnumerable<Gizmo> CompGetGizmosExtra()
		{
			var sw = new System.Diagnostics.Stopwatch();
			sw.Start();

			try
			{
				foreach (Gizmo g in base.CompGetGizmosExtra())
					yield return g;

				yield return ArgrillianGizmoHelpers.Toggle(
					"Medic",
					"ON: this pawn will triage/tend/haul injured allies.",
					() => isMedic,
					() =>
					{
						isMedic = !isMedic;

						// Turning medic on registers this pawn.
						if (isMedic)
						{
							if (parent is Pawn pawn) RegisterMedic(pawn);
						}
						else
						{
							combatMedic = false;
							assignedPawnThingID = -1;
							if (parent is Pawn pawn) UnregisterMedic(pawn);
						}
					}
				);

				yield return ArgrillianGizmoHelpers.Toggle(
					"Combat medic",
					"ON: protects patient by taking cover out of line of sight and only attacks when enemies are very close to the patient.",
					() => combatMedic,
					() =>
					{
						combatMedic = !combatMedic;

						// combat medic implies medic.
						if (combatMedic)
							isMedic = true;

						// If this pawn is already on the map, update the relevant registration.
						if (parent is Pawn pawn)
						{
							if (combatMedic)
								RegisterMedic(pawn);
							else
								UnregisterMedic(pawn);
						}

						// If toggling combat medic off, unassign it.
						if (!combatMedic)
							assignedPawnThingID = -1;
					}
				);
			}
			finally
			{
				sw.Stop();

				// Player.log only if slow (no spam). Threshold tuned for menu/startup lag.
				const double thresholdMs = 5.0;
				if (!loggedMedicGizmosOnce && sw.Elapsed.TotalMilliseconds >= thresholdMs)
				{
					loggedMedicGizmosOnce = true;
					Log.Message($"[ArgrillianThreat] CompGetGizmosExtra(MedicSettings) slow: {sw.Elapsed.TotalMilliseconds:0.00} ms parent={(parent != null ? parent.ToString() : "null")}");
				}
			}
		}

		public void ExposeData()
		{
			Scribe_Values.Look(ref isMedic, "isMedic", false);
			Scribe_Values.Look(ref combatMedic, "combatMedic", false);
			Scribe_Values.Look(ref assignedPawnThingID, "assignedPawnThingID", -1);

			// Safety normalization: combat medic implies medic.
			if (combatMedic)
				isMedic = true;
		}
	}

	// -----------------------------
	// Combat AI / Patient retreat
	// -----------------------------
	public class JobGiver_ArgrillianThreatResponse : ThinkNode_JobGiver
	{
		public float scanRange = 60f;
		public float assistAllyScanRange = 50f;

		public int minStepCooldownTicks = 60;

		public float desiredCombatDistance = 0.85f; // still used as scalar; band is weapon-derived in GetDesiredCombatDistance
		public float approachSlackMultiplier = 1.35f;
		public float meleeChaseMultiplier = 1.60f;
		public float meleeProtectRestrictMultiplier = 1.20f;

		public float losBreakBonus = 25f;

		public float retreatMinHealthPercent = 0.05f;
		public float retreatSearchRadius = 18f;

		private float meleePopOutDistanceFactor = 1.08f;
		private float rangedFiringBandFactor = 1.05f;

		// Patient retreat thresholds
		private float patientRetreatSafeDistanceFromHostile = 18f;
		public float patientRetreatSearchRadius = 28f;
		public float patientRetreatPreferMedicRadius = 70f;

		// Patient-like threshold
		public float patientRetreatMinHPPercentToTreatAsPatient = 0.75f;

		public int patientRetreatModeHysteresisTicks = 180;
		private float patientRetreatMinHPPercentToLockIn = 0.58f;

		private float rangedPursuitCloseFactor = 0.75f;
		private float rangedPursuitMinApproachMultiplier = 0.60f;

		// ---- combat medic aid tuning ----
		//private float combatMedicAidHPPercentThreshold = 0.75f;        // ally injured enough to trigger medic aid
		private float combatMedicAidMinRange = 25f;

		// --- NEW: injury thresholds for combat medics / patients ---
		private float combatMedicInjuredHPPercentThreshold = 0.85f;   // injured enough to stop engaging

		private float combatMedicAssistEngageDistanceMultiplier = 1.6f; // immediate threat heuristic
		private float immediateThreatScanRadius = 35f;                  // immediate threat heuristic

		// NEW: patient TEND override tuning
		//private int tendOverrideMinStableTicks = 10; // small buffer to reduce “fight vs tend” tug-of-war

		private float GetDesiredCombatDistance(Pawn pawn, Pawn hostile, bool pursueAdvance)
		{
			if (pawn == null || hostile == null) return 30f;
			if (pawn.Map == null || hostile.Map == null) return 30f;

			float pawnHP = pawn.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
			bool isPatientMode = pawnHP <= patientRetreatMinHPPercentToTreatAsPatient;

			var medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
			bool isCombatMedic = medicComp != null && medicComp.isMedic && medicComp.combatMedic;

			float meleeRange = GetPawnMeleeRange(pawn);
			float distNow = pawn.Position.DistanceTo(hostile.Position);

			if (meleeRange > 0f && distNow <= (meleeRange + 0.35f))
				return meleeRange + 0.25f;

			Verb v = pawn.TryGetAttackVerb(hostile);
			if (v == null) return 30f;

			if (v is Verb_Shoot shoot)
			{
				float min = shoot.verbProps.minRange;
				float max = shoot.verbProps.range;
				if (max <= 0.01f) return 30f;

				float want = max * desiredCombatDistance;
				want = Mathf.Max(want, min + 0.5f);
				want = Mathf.Min(want, max);

				if (isPatientMode || isCombatMedic)
				{
					want = Mathf.Lerp(min + 0.8f, want, 0.65f);
				}

				HostileMotionSample prev = UpdateAndGetMotion(pawn, hostile);
				bool havePrev = prev.initialized;

				if (havePrev)
				{
					Vector3 hostilePrevPos = prev.lastHostileCell.ToVector3Shifted();
					Vector3 hostileNowPos = hostile.Position.ToVector3Shifted();

					Vector3 relNow = hostileNowPos - pawn.Position.ToVector3Shifted();
					Vector3 relPrev = hostilePrevPos - pawn.Position.ToVector3Shifted();

					Vector3 relNowDir = Direction2D(relNow);
					Vector3 relPrevDir = Direction2D(relPrev);

					if (relNowDir != Vector3.zero && relPrevDir != Vector3.zero)
					{
						float prevDist = relPrev.magnitude;
						float currDist = relNow.magnitude;

						float distDelta = currDist - prevDist; // >0 hostile moved away, <0 hostile moved toward

						float norm = distDelta / 2.0f;
						norm = Mathf.Clamp(norm, -1.0f, 1.0f);

						float trendLoosenMultiplier = pursueAdvance ? 0.65f : 1.0f;

						if (norm > 0.01f)
						{
							float t = norm * trendLoosenMultiplier;
							float target = Mathf.Lerp(want, max, t);
							want = Mathf.Min(target, max);
						}
						else if (norm < -0.01f)
						{
							float t = (-norm);
							float target = Mathf.Lerp(want, min, t);
							want = Mathf.Max(target, min);
						}
					}
				}

				want = Mathf.Clamp(want, min + 0.01f, max);
				return want;
			}

			if (meleeRange > 0f)
				return meleeRange;

			return 2.2f;
		}

		private float GetPawnMeleeRange(Pawn pawn)
		{
			if (pawn == null) return 0f;

			float best = 0f;
			var vt = pawn.verbTracker;
			if (vt != null)
			{
				foreach (var verb in vt.AllVerbs)
				{
					if (verb is Verb_MeleeAttack melee)
					{
						float r = melee.verbProps?.range ?? 0f;
						if (r > best) best = r;
					}
				}
			}

			return best > 0f ? best : 2.2f;
		}

		private float GetEngageDistanceBandMultiplierForPawn(Pawn pawn)
		{
			return 1f;
		}

		private bool IsInjuredEnoughForCombatMedicToStopFighting(Pawn p)
		{
			if (p == null || p.Dead || p.health == null) return false;
			float hpPct = p.health.summaryHealth.SummaryHealthPercent;
			return hpPct <= combatMedicInjuredHPPercentThreshold;
		}

		private CompArgrillianThreatSettings Settings(Pawn pawn) => pawn?.GetComp<CompArgrillianThreatSettings>();

		private Pawn FindBestAidTargetForCombatMedic(Pawn medic, float maxRange)
		{
			if (medic == null || medic.Dead || medic.Map == null) return null;

			// EVENT-DRIVEN: Combat medics must not scan the map for patients.
			// The alert system maintains the active PatientCall set + prioritization (Bleed > downed).
			// This method becomes a pure consumer of that cached state.
			Pawn best = ArgrillianAlertSystem.GetBestPatientFromCalls(medic, maxRange);

			if (best == null) return null;
			if (best.Dead) return null;
			if (!best.Spawned || best.Map != medic.Map) return null;

			// Optional sanity: only accept if within the desired band.
			// (GetBestPatientFromCalls may already honor range, but this keeps behavior tight.)
			float d = medic.Position.DistanceTo(best.Position);
			if (d > maxRange) return null;

			return best;
		}

		private bool IsImmediateThreat(Pawn self, Pawn hostile, float desiredDistanceBand)
		{
			if (self == null || self.Dead || hostile == null || hostile.Dead) return false;
			if (self.Map == null || hostile.Map == null || self.Map != hostile.Map) return false;

			float d = self.Position.DistanceTo(hostile.Position);
			if (d > immediateThreatScanRadius) return false;

			// Engage band grows with desired distance, but has a floor to avoid "always false" at small bands.
			float engageBand = Mathf.Max(8f, desiredDistanceBand * combatMedicAssistEngageDistanceMultiplier);
			return d <= engageBand;
		}

		private bool TryPickRetreatCell(Pawn pawn, Pawn attacker, float desiredDistance, out IntVec3 bestCell)
		{
			bestCell = pawn.Position;
			Map map = pawn.Map;

			float bestScore = float.NegativeInfinity;

			bool avoidBeingInFriendlyLineOfFire = true;
			float friendlyLineToleranceTiles = 2.5f;
			float friendScanRadius = 25f;

			for (int i = 0; i < ArgrillianThreatGeometry.GetAdj8().Length; i++)
			{
				IntVec3 off = ArgrillianThreatGeometry.GetAdj8()[i];
				IntVec3 c = pawn.Position + off;

				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null) continue;
				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				if (avoidBeingInFriendlyLineOfFire &&
					IsCellInFriendlyLineOfFireForRetreat(pawn, attacker, c, friendlyLineToleranceTiles, friendScanRadius))
					continue;

				float d = c.DistanceTo(attacker.Position);

				float bandTooLow = Mathf.Max(0f, desiredDistance - d);
				float score = d - bandTooLow * 1.8f;

				if (score > bestScore)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			if (bestCell != pawn.Position)
				return true;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(pawn.Position, retreatSearchRadius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null) continue;
				if (!pawn.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				if (avoidBeingInFriendlyLineOfFire &&
					IsCellInFriendlyLineOfFireForRetreat(pawn, attacker, c, friendlyLineToleranceTiles, friendScanRadius))
					continue;

				float d = c.DistanceTo(attacker.Position);

				float bandTooLow = Mathf.Max(0f, desiredDistance - d);
				float score = d - bandTooLow * 2.0f;

				if (!GenSight.LineOfSight(c, pawn.Position, map))
					score += 2f;

				if (score > bestScore + 0.01f)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			return bestCell != pawn.Position;
		}

		private bool IsCellInFriendlyLineOfFireForRetreat(Pawn self, Pawn hostile, IntVec3 candidateCell, float friendlyLineToleranceTiles, float friendScanRadius)
		{
			if (self == null || hostile == null || self.Map == null) return false;

			Map map = self.Map;
			Vector3 candidate = candidateCell.ToVector3Shifted();

			float r = Mathf.Max(1f, friendScanRadius);

			foreach (IntVec3 c in GenRadial.RadialCellsAround(self.Position, r, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				foreach (Thing thing in c.GetThingList(map))
				{
					if (thing is not Pawn friend) continue;
					if (friend == self || friend.Dead) continue;
					if (friend.Faction != self.Faction) continue;

					Verb fv = friend.TryGetAttackVerb(hostile);
					if (fv == null || fv is not Verb_Shoot) continue;

					if (!GenSight.LineOfSight(friend.Position, hostile.Position, map)) continue;

					Vector3 shooter = friend.Position.ToVector3Shifted();
					Vector3 hostilePos = hostile.Position.ToVector3Shifted();

					Vector3 seg = hostilePos - shooter;
					float segLenSq = seg.sqrMagnitude;
					if (segLenSq < 0.001f) continue;

					float projT = Vector3.Dot(candidate - shooter, seg) / segLenSq;
					projT = Mathf.Clamp01(projT);

					Vector3 closest = shooter + seg * projT;
					float distToLine = Vector3.Distance(candidate, closest);

					if (distToLine <= friendlyLineToleranceTiles)
						return true;
				}
			}

			return false;
		}

		private struct HostileMotionSample
		{
			public IntVec3 lastHostileCell;
			public int lastTick;
			public bool initialized;
		}

		// NOTE: This cache is used only to smooth GetDesiredCombatDistance.
		// We prune aggressively to prevent unbounded growth on long-running saves.
		// Replace/add near the HostileMotion fields

		private static readonly Dictionary<(int attackerId, int hostileId, int mapId), HostileMotionSample> HostileMotion =
			new Dictionary<(int, int, int), HostileMotionSample>();

		private static readonly List<(int, int, int)> HostileMotionRemoveKeysBuffer = new List<(int, int, int)>(64);

		private static int HostileMotionMaxEntries = 4096;
		private static int HostileMotionEntryTtlTicks = 900;

		private static void HostileMotionPruneIfNeeded(int tick)
		{
			// Fast path: no pruning needed.
			int count = HostileMotion.Count;
			if (count <= HostileMotionMaxEntries * 0.75f)
			{
				if (count <= HostileMotionMaxEntries)
					return;

				// If it can ever be > max without hitting the 0.75 branch, we still clear.
				if (count <= HostileMotionMaxEntries) return;
			}

			if (count > HostileMotionMaxEntries)
			{
				HostileMotion.Clear();
				return;
			}

			// TTL prune (allocation-free)
			HostileMotionRemoveKeysBuffer.Clear();

			for (var it = HostileMotion.GetEnumerator(); it.MoveNext();)
			{
				var kvp = it.Current;
				if (!kvp.Value.initialized) continue;

				if (tick - kvp.Value.lastTick > HostileMotionEntryTtlTicks)
					HostileMotionRemoveKeysBuffer.Add(kvp.Key);
			}

			for (int i = 0; i < HostileMotionRemoveKeysBuffer.Count; i++)
				HostileMotion.Remove(HostileMotionRemoveKeysBuffer[i]);
		}

		private static readonly List<(int attackerId, int hostileId, int mapId)> HostileMotionPruneBuffer =
		new List<(int, int, int)>();

		private static HostileMotionSample UpdateAndGetMotion(Pawn attacker, Pawn hostile)
		{
			if (attacker == null || hostile == null || attacker.Map == null || hostile.Map == null)
				return default;

			int tick = Find.TickManager != null ? Find.TickManager.TicksGame : 0;

			// Very lightweight pruning to avoid unbounded dictionary growth.
			HostileMotionPruneIfNeeded(tick);

			var key = (attacker.thingIDNumber, hostile.thingIDNumber, attacker.Map.uniqueID);

			if (!HostileMotion.TryGetValue(key, out var sample) || !sample.initialized)
			{
				sample = new HostileMotionSample
				{
					lastHostileCell = hostile.Position,
					lastTick = tick,
					initialized = true
				};
				HostileMotion[key] = sample;
				return default;
			}

			var prev = sample;

			sample.lastHostileCell = hostile.Position;
			sample.lastTick = tick;
			sample.initialized = true;
			HostileMotion[key] = sample;

			return prev;
		}

		private static Vector3 Direction2D(Vector3 v)
		{
			v.y = 0f;
			if (v.sqrMagnitude < 0.000001f) return Vector3.zero;
			return v.normalized;
		}

		protected override Job TryGiveJob(Pawn pawn)
		{
			// NEW: publish PatientCalls when this pawn enters downed/bleeding states
			// (edge/transition coalescing is handled inside NotifyPawnSelfState).
			ArgrillianAlertSystem.NotifyPawnSelfState(pawn);
			// Medic gating: non-combat medics don't do threat response.
			var medicThreatSettings = pawn?.GetComp<CompArgrillianMedicSettings>();
			if (medicThreatSettings != null && medicThreatSettings.isMedic && !medicThreatSettings.combatMedic)
			{
				return null;
			}

			if (pawn == null || pawn.Dead || pawn.Map == null) return null;
			if (pawn.Downed) return null;

			var map = pawn.Map;

			if (!ArgrillianThreatHelpers.TryBuildContext(pawn, out ThreatContext tctx))
				return null;

			CompArgrillianThreatSettings s = tctx.Settings;

			bool pursueAdvance = s?.pursueAdvance ?? true;
			bool guardFellowPawns = s?.guardFellowPawns ?? true;
			bool squadMode = s?.squadMode ?? false;

			bool skipDecisionTick = ArgrillianThreatState.ThreatTickCache.ShouldWait(pawn, minStepCooldownTicks);
			bool committed = ArgrillianThreatState.CombatCommit.RecentlyCommitted(pawn);

			float pawnHP = pawn.health?.summaryHealth?.SummaryHealthPercent ?? 1f;

			var acquireCtx = new ArgrillianThreatHostileAcquireContext
			{
				Map = map,
				ScanRange = scanRange,
				AssistAllyScanRange = assistAllyScanRange,
				SquadMode = squadMode,
				GuardFellowPawns = guardFellowPawns
			};

			if (!ArgrillianThreatTargeting.TryAcquireHostileFromExistingSystems(pawn, tctx, acquireCtx, out HostileContext hctx))
				return null;

			Pawn hostile = hctx.Hostile;

			if (hostile == null || hostile.Dead)
			{
				if (hostile != null)
					ArgrillianAlertSystem.NotifyPawnHostileEliminated(pawn, hostile);

				ArgrillianThreatState.CombatLock.Clear(pawn);
				ArgrillianThreatState.CombatCommit.Clear(pawn);
				return null;
			}

			if (!hostile.Spawned || hostile.Map == null || hostile.Map != map ||
				!hostile.Position.InBounds(map) ||
				!pawn.Spawned)
			{
				ArgrillianThreatState.CombatLock.Clear(pawn);
				ArgrillianThreatState.CombatCommit.Clear(pawn);
				return null;
			}

			ArgrillianThreatState.CombatLock.MarkSeen(pawn, hostile);
			ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);

			int prevHostileId = ArgrillianThreatState.AwarenessCache.GetLastKnownHostileId(pawn);
			IntVec3 prevCell = ArgrillianThreatState.AwarenessCache.GetLastKnownCell(pawn);

			ArgrillianThreatState.AwarenessCache.MarkDirect(pawn, hostile);

			int hostileId = hostile.thingIDNumber;
			IntVec3 newCell = hostile.Position;

			bool shouldBroadcast =
				prevHostileId != hostileId
				|| prevCell != newCell;

			if (shouldBroadcast)
			{
				ArgrillianAlertSystem.BroadcastSharedAwareness(
					source: pawn,
					hostile: hostile,
					allyRadius: 30f,
					squadOnly: true
				);
			}

			bool wantsPatientRetreat;
			byte currentMode = ArgrillianThreatMode.UpdateMode(
				pawn,
				pawnHP,
				patientRetreatMinHPPercentToTreatAsPatient,
				patientRetreatMinHPPercentToLockIn,
				patientRetreatModeHysteresisTicks,
				ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
				out wantsPatientRetreat
			);

			float desiredCombatDistanceNow = GetDesiredCombatDistance(pawn, hostile, pursueAdvance);

			var medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
			bool isCombatMedic = medicComp != null && medicComp.isMedic && medicComp.combatMedic;

			// =========================
			// COMBAT MEDIC ASSIST MODE
			// =========================
			if (isCombatMedic)
			{
				// FIX: if the medic themself is injured enough, stop fighting immediately
				// so their job arbitration can transition into tending/rescue.
				if (IsInjuredEnoughForCombatMedicToStopFighting(pawn))
				{
					Pawn nearestMedic = ArgrillianThreatExecution.FindNearestMedic(pawn);

					return ArgrillianThreatExecution.ExecutePatientRetreat(
						pawn,
						tctx,
						hctx,
						desiredCombatDistanceNow,
						pursueAdvance: false,
						skipDecisionTick: skipDecisionTick,
						nearestMedic: nearestMedic,
						patientRetreatSafeDistanceFromHostile,
						patientRetreatSearchRadius,
						patientRetreatPreferMedicRadius,
						patientRetreatMinHPPercentToTreatAsPatient,
						patientRetreatMinHPPercentToLockIn,
						patientRetreatModeHysteresisTicks,
						ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
						retreatMinHealthPercent,
						losBreakBonus,
						scanRange
					);
				}

				Pawn assignedPatient = medicComp?.AssignedPawn;
				Pawn aidTarget = FindBestAidTargetForCombatMedic(pawn, combatMedicAidMinRange);

				bool hasRelevantPatientAssistTarget =
					aidTarget != null &&
					(assignedPatient != null ? aidTarget == assignedPatient : true);

				if (hasRelevantPatientAssistTarget)
				{
					if (aidTarget != null)
					{
						bool aidTargetTooInjuredToFight = IsInjuredEnoughForCombatMedicToStopFighting(aidTarget);

						// Treat downed as “stop fighting & get them safe” immediately.
						if (aidTarget.Downed || aidTargetTooInjuredToFight)
						{
							Pawn nearestMedic = ArgrillianThreatExecution.FindNearestMedic(pawn);

							return ArgrillianThreatExecution.ExecutePatientRetreat(
								pawn,
								tctx,
								hctx,
								desiredCombatDistanceNow,
								pursueAdvance: false,
								skipDecisionTick: skipDecisionTick,
								nearestMedic: nearestMedic,
								patientRetreatSafeDistanceFromHostile,
								patientRetreatSearchRadius,
								patientRetreatPreferMedicRadius,
								patientRetreatMinHPPercentToTreatAsPatient,
								patientRetreatMinHPPercentToLockIn,
								patientRetreatModeHysteresisTicks,
								ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
								retreatMinHealthPercent,
								losBreakBonus,
								scanRange
							);
						}

						// If the patient is in immediate danger, the medic should break off to create tend conditions.
						bool patientImmediateThreat = IsImmediateThreat(aidTarget, hostile, desiredCombatDistanceNow);
						if (patientImmediateThreat)
						{
							Pawn nearestMedic = ArgrillianThreatExecution.FindNearestMedic(pawn);

							return ArgrillianThreatExecution.ExecutePatientRetreat(
								pawn,
								tctx,
								hctx,
								desiredCombatDistanceNow,
								pursueAdvance: false,
								skipDecisionTick: skipDecisionTick,
								nearestMedic: nearestMedic,
								patientRetreatSafeDistanceFromHostile,
								patientRetreatSearchRadius,
								patientRetreatPreferMedicRadius,
								patientRetreatMinHPPercentToTreatAsPatient,
								patientRetreatMinHPPercentToLockIn,
								patientRetreatModeHysteresisTicks,
								ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
								retreatMinHealthPercent,
								losBreakBonus,
								scanRange
							);
						}
					}
				}
			}

			// =========================
			// PATIENT RETREAT (injured pawns)
			// =========================
			bool shouldRetreat = wantsPatientRetreat || (pawnHP <= retreatMinHealthPercent);

			if (shouldRetreat && !isCombatMedic)
			{
				Pawn nearestMedic = ArgrillianThreatExecution.FindNearestMedic(pawn);

				return ArgrillianThreatExecution.ExecutePatientRetreat(
					pawn,
					tctx,
					hctx,
					desiredCombatDistanceNow,
					pursueAdvance: false,
					skipDecisionTick: skipDecisionTick,
					nearestMedic: nearestMedic,
					patientRetreatSafeDistanceFromHostile,
					patientRetreatSearchRadius,
					patientRetreatPreferMedicRadius,
					patientRetreatMinHPPercentToTreatAsPatient,
					patientRetreatMinHPPercentToLockIn,
					patientRetreatModeHysteresisTicks,
					ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
					retreatMinHealthPercent,
					losBreakBonus,
					scanRange
				);
			}

			bool retreatLockout = ArgrillianThreatState.FightLockoutAfterRetreat.RecentlyEndedRetreat(
				pawn,
				ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks
			);

			if (retreatLockout && pawn.CurJob != null)
			{
				if (pawn.CurJob.def == JobDefOf.TendPatient || pawn.CurJob.def == JobDefOf.Rescue)
					return pawn.CurJob;

				if (pawn.CurJob.def == JobDefOf.Goto)
					return pawn.CurJob;
			}

			if (skipDecisionTick)
			{
				if (pawn.CurJob != null)
				{
					if (pawn.CurJob.def == JobDefOf.AttackMelee || pawn.CurJob.def == JobDefOf.AttackStatic)
						return pawn.CurJob;

					if (pawn.CurJob.def == JobDefOf.Goto && !pursueAdvance)
						return null;

					if (committed)
						return pawn.CurJob;

					Pawn locked = ArgrillianThreatState.CombatLock.TryGetLockedHostile(pawn);
					if (locked != null)
					{
						// fall through
					}
				}

				bool useImmediateHardGate = shouldRetreat || wantsPatientRetreat;

				if (useImmediateHardGate)
				{
					bool immediateThreat = IsImmediateThreat(pawn, hostile, desiredCombatDistanceNow);
					if (!immediateThreat)
						return null;
				}

				return ArgrillianThreatExecution.ExecuteFightMode(
					pawn,
					tctx,
					hctx,
					desiredCombatDistanceNow,
					pursueAdvance,
					guardFellowPawns,
					squadMode,
					skipAggressiveStart: false,
					losBreakBonus,
					retreatMinHealthPercent,
					minStepCooldownTicks,
					scanRange,
					approachSlackMultiplier,
					meleeChaseMultiplier,
					meleeProtectRestrictMultiplier,
					meleePopOutDistanceFactor,
					rangedPursuitCloseFactor,
					rangedPursuitMinApproachMultiplier,
					rangedFiringBandFactor,
					ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks
				);
			}

			bool alerted = ArgrillianThreatState.AwarenessCache.IsHighAlert(pawn) || ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(pawn);

			if (alerted && !skipDecisionTick)
			{
				bool hasValidHostile =
					(hostile != null && !hostile.Dead && hostile.Spawned && hostile.Map == map);

				if (!hasValidHostile)
				{
					// If we were tracking a hostile but it’s now eliminated/invalid for this map, cancel it once.
					if (hostile != null)
						ArgrillianAlertSystem.NotifyPawnHostileEliminated(pawn, hostile);

					// Fall through: your existing logic will handle no-LOS/no-op behavior.
				}
				else
				{
					bool hasLOS = GenSight.LineOfSight(pawn.Position, hostile.Position, map);



					// Edge publish:
					// - Seen => tell alert system
					// - Lost sight => tell alert system "investigate last known"
					// We use pawn’s current position as “caller”; lastKnownCell is stable per hostile position at publish moment.
					if (hasLOS)
					{
						ArgrillianAlertSystem.NotifyPawnSeesHostile(pawn, hostile, hostile.Position);
					}
					else
					{
						ArgrillianAlertSystem.NotifyPawnLostSightOfHostile(pawn, hostile, hostile.Position);
					}

					if (hasLOS)
					{
						bool immediateThreat = IsImmediateThreat(pawn, hostile, desiredCombatDistanceNow);

						bool useImmediateHardGate = isCombatMedic || shouldRetreat || wantsPatientRetreat;

						if (!immediateThreat && useImmediateHardGate)
						{
							Pawn nearestMedic = ArgrillianThreatExecution.FindNearestMedic(pawn);
							return ArgrillianThreatExecution.ExecutePatientRetreat(
								pawn,
								tctx,
								hctx,
								desiredCombatDistanceNow,
								pursueAdvance: false,
								skipDecisionTick: skipDecisionTick,
								nearestMedic: nearestMedic,
								patientRetreatSafeDistanceFromHostile,
								patientRetreatSearchRadius,
								patientRetreatPreferMedicRadius,
								patientRetreatMinHPPercentToTreatAsPatient,
								patientRetreatMinHPPercentToLockIn,
								patientRetreatModeHysteresisTicks,
								ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks,
								retreatMinHealthPercent,
								losBreakBonus,
								scanRange
							);
						}

						if (immediateThreat)
						{
							Job alertedJob = ArgrillianThreatExecution.ExecuteAlertedMode(
								pawn,
								hostileIfAny: hctx.Hostile,
								desiredCombatDistanceNow: desiredCombatDistanceNow,
								losBreakBonus: losBreakBonus,
								isRanged: hctx.IsRanged,
								minHighAlertTicksToMove: minStepCooldownTicks,
								scanRange: scanRange,
								retreatMinHealthPercent: retreatMinHealthPercent,
								lastKnownInvestigateRadius: 12f
							);

							if (alertedJob != null)
								return alertedJob;
						}
					}
				}
			}

			if (shouldRetreat)
			{
				ArgrillianThreatState.CombatCommit.Clear(pawn);

				if (TryPickRetreatCell(pawn, hostile, desiredCombatDistanceNow, out IntVec3 retreatCell))
				{
					var keep = ArgrillianGotoHelper.KeepIfSameGoto(pawn, retreatCell);
					if (keep != null) return keep;

					ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
					return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, retreatCell);
				}

				var keep2 = ArgrillianGotoHelper.KeepIfSameGoto(pawn, pawn.Position);
				if (keep2 != null) return keep2;

				ArgrillianThreatState.ThreatTickCache.MarkNow(pawn);
				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, pawn.Position);
			}

			if (currentMode == 1)
			{
				if (!ArgrillianThreatState.FightLockoutAfterRetreat.RecentlyEndedRetreat(
						pawn,
						ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks))
					ArgrillianThreatState.FightLockoutAfterRetreat.MarkRetreatEndAttempt(pawn);
			}

			bool finalUseImmediateHardGate = shouldRetreat || wantsPatientRetreat;

			// NEW: if we have no LOS and we're not in immediate threat,
			// don't keep planning fight-mode movement (prevents "keep chasing after enemies are gone").
			if (!finalUseImmediateHardGate)
			{
				bool hasLOS = (hostile != null && !hostile.Dead && hostile.Spawned && hostile.Map == map)
					&& GenSight.LineOfSight(pawn.Position, hostile.Position, map);

				bool finalImmediateThreat = IsImmediateThreat(pawn, hostile, desiredCombatDistanceNow);

				if (!hasLOS && !finalImmediateThreat)
					return null;
			}

			if (finalUseImmediateHardGate)
			{
				bool finalImmediateThreat = IsImmediateThreat(pawn, hostile, desiredCombatDistanceNow);
				if (!finalImmediateThreat)
					return null;
			}

			return ArgrillianThreatExecution.ExecuteFightMode(
				pawn,
				tctx,
				hctx,
				desiredCombatDistanceNow,
				pursueAdvance,
				guardFellowPawns,
				squadMode,
				skipAggressiveStart: retreatLockout,
				losBreakBonus,
				retreatMinHealthPercent,
				minStepCooldownTicks,
				scanRange,
				approachSlackMultiplier,
				meleeChaseMultiplier,
				meleeProtectRestrictMultiplier,
				meleePopOutDistanceFactor,
				rangedPursuitCloseFactor,
				rangedPursuitMinApproachMultiplier,
				rangedFiringBandFactor,
				ArgrillianThreatTuning.PatientFightLockoutAfterRetreatTicks);
		}
	}

	// -----------------------------
	// Medic/Combat Medic AI
	// -----------------------------
	public class JobGiver_TendRetreatingAllies : ThinkNode_JobGiver
	{
		public float treatBelowHealthPercent = 0.65f;
		public float searchRadius = 70f;

		public float urgentTendHealthPercent = 0.35f;
		public float severeBleedHediffSeverity = 0.15f;

		// NEW: give combat medics a higher/earlier stop-tending threshold for their assigned patient
		public float combatMedicInjuredHPPercentThreshold = 0.85f;

		public int tendTaskStickinessTicks = 120;

		// How close combat medics try to be
		public float combatTendMaxDistance = 10f;

		public bool stopPatientWhenUrgent = true;

		public int minJobSwitchCooldownTicks = 30;

		public bool disableTacticsIfHostilesPresent = true;

		public float hospitalBedMaxDist = 70f;

		// NEW: require stability for tending
		public int patientStableTicksRequired = 18;

		// NEW: prevent repeated "fall out of bed -> rescue -> fall out again" loops
		// (only blocks Rescue transitions; medic can still tend while not stable enough)
		public int combatMedicRescueStableTicksRequired = 24;

		// NEW: score-based urgency to reliably catch "dying soon" bloodloss states
		public float urgentScoreThreshold = 180f;

		// NEW: prevent "move -> rescue -> move" oscillation for the same patient
		public int combatMedicRescueAttemptCooldownTicks = 90;

		// NEW: strong bias so combat medics pick downed pawns over merely-injured chasers
		public float downedPatientScoreBonus = 800f;

		private CompArgrillianMedicSettings Settings(Pawn pawn) => pawn?.GetComp<CompArgrillianMedicSettings>();

		private static readonly System.Collections.Generic.Dictionary<int, int> lastTendLockSwitchTick = new System.Collections.Generic.Dictionary<int, int>();

		private static int TendNow => Find.TickManager.TicksGame;

		private static int TendSwitchKey(Pawn medic, Pawn patient)
		{
			unchecked
			{
				return (medic.thingIDNumber * 397) ^ patient.thingIDNumber;
			}
		}

		// Returns true if we should NOT switch locks yet (cooldown still active).
		private static bool RecentlySwitchedTendLock(Pawn medic, Pawn patient, int cooldownTicks)
		{
			if (medic == null || patient == null) return false;

			int now = TendNow;
			int k = TendSwitchKey(medic, patient);

			if (!lastTendLockSwitchTick.TryGetValue(k, out int t))
				return false;

			if (t > now) return false;
			return (now - t) < cooldownTicks;
		}

		private static void MarkSwitchedTendLock(Pawn medic, Pawn patient)
		{
			if (medic == null || patient == null) return;
			lastTendLockSwitchTick[TendSwitchKey(medic, patient)] = TendNow;
		}

		private static bool HasBloodLossStatic(Pawn p)
		{
			return p?.health?.hediffSet != null
				&& p.health.hediffSet.HasHediff(HediffDefOf.BloodLoss);
		}

		private static float BloodLossSeverityStatic(Pawn p)
		{
			if (p?.health?.hediffSet == null) return 0f;
			var hd = p.health.hediffSet.GetFirstHediffOfDef(HediffDefOf.BloodLoss);
			return hd?.Severity ?? 0f;
		}

		private static float PatientUrgencyStatic(Pawn patient, float treatBelowPercent)
		{
			if (patient?.health == null) return 0f;

			float hpPct = patient.health.summaryHealth.SummaryHealthPercent;
			float bleeding = HasBloodLossStatic(patient) ? BloodLossSeverityStatic(patient) : 0f;

			float urgency = (1f - hpPct) * 300f + bleeding * 35f;
			if (patient.Downed) urgency += 50f;
			if (hpPct <= treatBelowPercent) urgency += 40f;
			return urgency;
		}

		private static bool IsUrgentByScoreStatic(Pawn patient, float treatBelowPercent, float urgentThreshold)
		{
			float score = PatientUrgencyStatic(patient, treatBelowPercent);
			return score >= urgentThreshold;
		}

		private static bool CanReserveTendTarget(Pawn medic, Pawn patient)
		{
			if (medic == null || patient == null) return false;
			if (patient.Dead) return false;
			if (medic.Downed) return false;
			if (!medic.Spawned || medic.Map == null) return false;
			if (patient.Map == null || patient.Map != medic.Map) return false;

			return medic.CanReserve(patient, 1, -1, null, false);
		}

		private static bool IsMedicineThing(Thing thing)
		{
			if (thing == null) return false;
			if (thing.def == null) return false;

			// RimWorld 1.6: ThingDef has IsMedicine for medical items.
			if (thing.def.IsMedicine) return true;

			return false;
		}

		private Thing FindBestMedicineForTend(Pawn medic, Pawn patient, float radius)
		{
			if (medic == null || patient == null) return null;

			Map map = medic.Map;
			if (map == null || patient.Map != map) return null;

			// Medicine already on medic (instant, no travel)
			if (medic.carryTracker != null)
			{
				Thing carried = medic.carryTracker.CarriedThing;
				if (carried != null && IsMedicineThing(carried))
					return carried;
			}

			// Medicine in medic inventory (instant, no travel)
			if (medic.inventory != null)
			{
				foreach (Thing t in medic.inventory.innerContainer)
				{
					if (!IsMedicineThing(t)) continue;
					return t;
				}
			}

			IntVec3 patientCenter = patient.Position;

			// Battlefield medicine selection constraint:
			// - immediate area / bounded search only
			// - include medicine dropped/held by corpses in that area
			// - ignore forbidden timing reliability (don’t hard-gate on forbidden)
			// - hard cap the max radius so RimWorld never “upscales” into long-range fetch behavior
			//
			// If we return null, RimWorld may fall back into its own medicine-fetch routing.
			// So we do a slightly wider bounded second pass (still clamped).
			float rBase = Mathf.Max(radius, radius * 1.15f);
			if (rBase < 12f) rBase = 12f;

			float rCap = 18f;
			float r1 = Mathf.Min(rBase, rCap);

			Thing best = FindBestLocalMedicineInRadius(map, patientCenter, r1, includeCorpseHeld: true);
			if (best != null) return best;

			// Second bounded pass (still clamped; never becomes long-range).
			float r2 = Mathf.Min(rCap, r1 + 3f);
			if (r2 > r1)
			{
				best = FindBestLocalMedicineInRadius(map, patientCenter, r2, includeCorpseHeld: true);
				return best;
			}

			return null;
		}

		private Thing FindBestLocalMedicineInRadius(Map map, IntVec3 center, float radius, bool includeCorpseHeld)
		{
			float bestDist = float.PositiveInfinity;
			Thing best = null;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(center, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				float d = c.DistanceTo(center);

				foreach (Thing t in c.GetThingList(map))
				{
					if (IsMedicineThing(t))
					{
						if (d < bestDist)
						{
							bestDist = d;
							best = t;
						}
					}

					if (includeCorpseHeld)
					{
						Corpse corpse = t as Corpse;
						if (corpse == null) continue;

						foreach (Thing inner in corpse.GetDirectlyHeldThings())
						{
							if (inner == null) continue;
							if (!IsMedicineThing(inner)) continue;

							if (d < bestDist)
							{
								bestDist = d;
								best = inner;
							}
						}
					}
				}
			}

			return best;
		}

		private Thing FindBestLocalMedicineForTendInRadius(Pawn medic, Map map, IntVec3 patientCenter, float radius)
		{
			float bestDist = float.PositiveInfinity;
			Thing best = null;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(patientCenter, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				float d = c.DistanceTo(patientCenter);

				foreach (Thing t in c.GetThingList(map))
				{
					// Directly on the ground / in containers surfaced by this cell list.
					// Intentionally do NOT filter by forbidden/unforbid state here:
					// the goal is to allow medicine recovery even if forbidden timing is unreliable.
					if (IsMedicineThing(t))
					{
						if (d < bestDist)
						{
							bestDist = d;
							best = t;
						}
					}

					// Medicine "dropped in the immediate area" held by corpses.
					Corpse corpse = t as Corpse;
					if (corpse != null)
					{
						foreach (Thing inner in corpse.GetDirectlyHeldThings())
						{
							if (inner == null) continue;
							if (!IsMedicineThing(inner)) continue;

							// Same rule: ignore forbidden timing here; just select something local.
							if (d < bestDist)
							{
								bestDist = d;
								best = inner;
							}
						}
					}
				}
			}

			return best;
		}

		private static bool IsMedicineFetchJob(Job j)
		{
			if (j == null || j.def == null) return false;

			string dn = j.def.defName;
			if (string.IsNullOrEmpty(dn)) return false;

			bool likelyFetch =
				IsHaulJob(j) ||
				dn.IndexOf("grab", System.StringComparison.OrdinalIgnoreCase) >= 0 ||
				dn.IndexOf("pickup", System.StringComparison.OrdinalIgnoreCase) >= 0 ||
				dn.IndexOf("carry", System.StringComparison.OrdinalIgnoreCase) >= 0;

			if (!likelyFetch) return false;

			if (j.targetA.IsValid && j.targetA.Thing != null && IsMedicineThing(j.targetA.Thing)) return true;
			if (j.targetB.IsValid && j.targetB.Thing != null && IsMedicineThing(j.targetB.Thing)) return true;
			if (j.targetC.IsValid && j.targetC.Thing != null && IsMedicineThing(j.targetC.Thing)) return true;

			return false;
		}

		private static readonly Dictionary<int, int> medicThingIdByPatientId = new Dictionary<int, int>();
		private static readonly Dictionary<int, int> medicCacheLastTickByPatientId = new Dictionary<int, int>();
		private static int medicCacheTtlTicks = 250; // ~4 sec at 60 ticks/sec

		private static Pawn FindAssignedCombatMedicForPatient(Pawn patient)
		{
			if (patient == null || patient.Dead || patient.Map == null)
				return null;

			Map map = patient.Map;
			int patientId = patient.thingIDNumber;

			// Fast path: cached mapping if still fresh.
			if (medicThingIdByPatientId.TryGetValue(patientId, out int medicId))
			{
				int lastTick = 0;
				medicCacheLastTickByPatientId.TryGetValue(patientId, out lastTick);

				int now = Find.TickManager != null ? Find.TickManager.TicksGame : 0;
				if (now - lastTick <= medicCacheTtlTicks)
				{
					// Avoid map-wide scans. Confirm medic id via cached combat medics only.
					if (medicId >= 0)
					{
						var combatMedics = CompArgrillianMedicSettings.GetCombatMedics(map);
						if (combatMedics != null)
						{
							for (int i = 0; i < combatMedics.Count; i++)
							{
								Pawn medic = combatMedics[i];
								if (medic == null || medic.Dead) continue;
								if (!medic.Spawned || medic.Map != map) continue;

								if (medic.thingIDNumber == medicId)
									return medic;
							}
						}
					}

					// Cached id stale/gone => fall through to refresh.
				}
			}

			// Refresh: iterate only cached combat medics on this map (no AllPawnsSpawned / no spawnedThings).
			Pawn found = null;

			var combatMedics2 = CompArgrillianMedicSettings.GetCombatMedics(map);
			if (combatMedics2 != null && combatMedics2.Count > 0)
			{
				for (int i = 0; i < combatMedics2.Count; i++)
				{
					Pawn medic = combatMedics2[i];
					if (medic == null || medic.Dead) continue;
					if (!medic.Spawned || medic.Map != map) continue;
					if (patient.Faction != null && medic.Faction != patient.Faction) continue;

					var medicComp = medic.GetComp<CompArgrillianMedicSettings>();
					if (medicComp == null || !medicComp.isMedic || !medicComp.combatMedic) continue;

					// assignment is already stored on the medic
					if (medicComp.assignedPawnThingID == patientId)
					{
						found = medic;
						break;
					}
				}
			}

			// Update cache (cache null as -1 to avoid repeated refresh within TTL).
			medicThingIdByPatientId[patientId] = found != null ? found.thingIDNumber : -1;
			medicCacheLastTickByPatientId[patientId] = Find.TickManager != null ? Find.TickManager.TicksGame : 0;

			return found;
		}

		private static bool IsValidTendTarget(Pawn patient, Pawn medic)
		{
			if (patient == null || medic == null) return false;
			if (patient.Dead) return false;

			PathEndMode endMode = patient.Downed ? PathEndMode.ClosestTouch : PathEndMode.Touch;
			return medic.CanReach(patient, endMode, Danger.Some);
		}

		private static bool CanReserveThing(Pawn medic, Thing t)
		{
			if (medic == null || t == null) return false;
			if (!medic.Spawned || medic.Map == null) return false;
			if (t.Map == null || t.Map != medic.Map) return false;

			return medic.CanReserve(t, 1, -1, null, false);
		}

		private static bool IsAllowedDownPawnJob(Job job)
		{
			if (job == null) return false;

			if (job.def == JobDefOf.Wait) return true;
			if (job.def == JobDefOf.LayDown) return true;
			if (job.def == JobDefOf.Rescue) return true;
			if (job.def == JobDefOf.TendPatient) return true;

			string defName = job.def?.defName;
			if (string.IsNullOrEmpty(defName)) return false;

			defName = defName.ToLowerInvariant();

			if (defName.Contains("rescue")) return true;
			if (defName.Contains("lay")) return true;
			if (defName.Contains("bed")) return true;

			return false;
		}

		private static void TryStopPatientToAllowTend(Pawn patient)
		{
			if (patient == null || patient.Dead) return;

			Job cur = patient.CurJob;
			if (cur == null) return;

			if (patient.Downed)
			{
				if (IsAllowedDownPawnJob(cur))
					return;

				// Patient is downed but on a non-tendable job:
				// Use StopAll(true) + clear queued jobs to prevent stand/haul oscillation
				// until the medic's Tend/Rescue pipeline completes.
				patient.jobs?.StopAll(true);
				patient.jobs?.ClearQueuedJobs();
				patient.pather?.StopDead();
				return;
			}
		}

		private static void TryStopPatientToAllowTendIfChasingOrAttacking(Pawn patient)
		{
			if (patient == null || patient.Dead) return;
			if (patient.Downed) return;

			Job j = patient.CurJob;
			if (j == null || j.def == null) return;

			if (IsCombatAttackLikeJob(j) || IsChaseOrTacticJob(j) || IsFleeLikeJob(j))
			{
				patient.jobs?.StopAll(true);
				patient.pather?.StopDead();
			}
		}

		private static void TryStopPatientToAllowTendIfUrgentNonDowned(Pawn patient)
		{
			if (patient == null || patient.Dead) return;
			if (patient.Downed) return;

			Job j = patient.CurJob;
			if (j == null || j.def == null) return;

			if (IsMealOrConsumeLikeJob(j))
			{
				patient.jobs?.StopAll(true);
				return;
			}

			if (IsHaulJob(j) ||
				(IsJobNameContains(j, "grab") || IsJobNameContains(j, "pickup") || IsJobNameContains(j, "carry")))
			{
				patient.jobs?.StopAll(true);
				return;
			}

			if (IsCombatAttackLikeJob(j) || IsChaseOrTacticJob(j) || IsFleeLikeJob(j))
			{
				patient.jobs?.StopAll(true);
				patient.pather?.StopDead();
				return;
			}

			// NEW: prevent urgent non-downed patients from walking off to LayDown while a medic is trying to tend them.
			if (j.def == JobDefOf.LayDown)
			{
				patient.jobs?.StopAll(true);
				return;
			}

			string defName = j.def.defName;
			if (!string.IsNullOrEmpty(defName))
			{
				defName = defName.ToLowerInvariant();

				if (defName.IndexOf("laydown", StringComparison.OrdinalIgnoreCase) >= 0 ||
					defName.IndexOf("lay_down", StringComparison.OrdinalIgnoreCase) >= 0)
				{
					patient.jobs?.StopAll(true);
					return;
				}

				// Extra conservative catch: only if the job name clearly implies “lay on/bed”.
				if (defName.IndexOf("bed", StringComparison.OrdinalIgnoreCase) >= 0 &&
					defName.IndexOf("lay", StringComparison.OrdinalIgnoreCase) >= 0)
				{
					patient.jobs?.StopAll(true);
					return;
				}
			}
		}

		private static bool IsJobNameContains(Job j, string part)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			return j.def.defName.IndexOf(part, StringComparison.OrdinalIgnoreCase) >= 0;
		}

		private static bool IsInterferingJobForTend(Pawn patient)
		{
			if (patient == null) return false;

			Job j = patient.CurJob;
			if (j == null) return false;

			if (patient.Downed && IsAllowedDownPawnJob(j))
				return false;

			if (!patient.Downed)
				return false;

			// NEW: if downed pawn is crawling/moving to safety, treat it as interfering so medic can stop it for tending/rescue.
			if (IsCrawlLikeJob(j) || IsMoveLikeJob(j))
				return true;

			// NEW: if downed pawn is on combat/chase behavior, treat it as interfering so we can stop it for tending.
			if (IsCombatAttackLikeJob(j) || IsChaseOrTacticJob(j))
				return true;

			if (IsHaulJob(j)) return true;

			string defName = j.def?.defName;
			if (string.IsNullOrEmpty(defName)) return false;

			if (defName.IndexOf("eat", StringComparison.OrdinalIgnoreCase) >= 0) return true;
			if (defName.IndexOf("ingest", StringComparison.OrdinalIgnoreCase) >= 0) return true;
			if (defName.IndexOf("meal", StringComparison.OrdinalIgnoreCase) >= 0) return true;
			if (defName.IndexOf("consume", StringComparison.OrdinalIgnoreCase) >= 0) return true;

			if (defName.IndexOf("rest", StringComparison.OrdinalIgnoreCase) >= 0) return true;

			if (defName.IndexOf("grab", StringComparison.OrdinalIgnoreCase) >= 0) return true;
			if (defName.IndexOf("pickup", StringComparison.OrdinalIgnoreCase) >= 0) return true;

			return false;
		}

		private static bool IsHaulJob(Job job)
		{
			if (job == null) return false;

			string[] candidates =
			{
				"Haul",
				"HaulToCell",
				"HaulToContainer",
				"HaulAI"
			};

			foreach (string name in candidates)
			{
				JobDef jd = DefDatabase<JobDef>.GetNamedSilentFail(name);
				if (jd != null && job.def == jd)
					return true;
			}

			string defName = job.def?.defName;
			if (string.IsNullOrEmpty(defName)) return false;

			defName = defName.ToLowerInvariant();

			// Original: haul-like
			if (defName.IndexOf("haul", StringComparison.OrdinalIgnoreCase) >= 0)
				return true;

			// RimWorld transport commonly appears as carry-to-* jobs (so treat those as haul-like too)
			// to prevent medics from bouncing into transport while the patient is downed/urgent.
			if (defName.IndexOf("carry", StringComparison.OrdinalIgnoreCase) >= 0)
				return true;

			return false;
		}

		private static bool IsJobRelevantToCombatMedicUrgent(Job job, Pawn patient)
		{
			if (job == null || patient == null) return false;

			if (job.def == JobDefOf.Rescue)
			{
				if (job.targetA.IsValid && job.targetA.Thing == patient) return true;
				if (job.targetB.IsValid && job.targetB.Thing == patient) return true;
				if (job.targetC.IsValid && job.targetC.Thing == patient) return true;
			}

			if (job.def == JobDefOf.TendPatient)
			{
				if (job.targetA.IsValid && job.targetA.Thing == patient) return true;
				if (job.targetB.IsValid && job.targetB.Thing == patient) return true;
				if (job.targetC.IsValid && job.targetC.Thing == patient) return true;
			}

			return false;
		}

		private static bool IsJobTargetValidForPatient(Job job, Pawn patient)
		{
			if (job == null || patient == null) return true;

			if (job.targetA.IsValid && job.targetA.Thing == patient)
			{
				if (patient.Dead) return false;
				if (patient.Map == null) return false;
			}

			return true;
		}

		private Building_Bed FindBestBedFor(Pawn medic, Pawn patient)
		{
			if (medic?.Map == null || patient?.Map == null) return null;
			if (medic.Map != patient.Map) return null;
			if (patient.Dead) return null;
			if (medic.Dead) return null;

			Map map = medic.Map;
			Building_Bed best = null;
			float bestScore = float.NegativeInfinity;

			foreach (var bed in map.listerBuildings.AllBuildingsColonistOfClass<Building_Bed>())
			{
				if (bed == null || bed.Destroyed || !bed.Spawned) continue;
				if (bed.Map != map) continue;

				float dist = medic.Position.DistanceTo(bed.Position);
				if (dist > hospitalBedMaxDist) continue;

				if (!medic.CanReach(bed.Position, PathEndMode.ClosestTouch, Danger.Some)) continue;

				if (!CanReserveThingForPatient(medic, bed, patient)) continue;

				float score = 0f;
				if (bed.Medical) score += 1200f;

				score += (hospitalBedMaxDist - dist) * 2f;

				float patientToBedDist = patient.Position.DistanceTo(bed.Position);
				score += (60f - patientToBedDist) * 0.2f;

				if (score > bestScore)
				{
					bestScore = score;
					best = bed;
				}
			}

			return best;
		}

		private Pawn FindNearestHostile(Pawn medic, float radius = 80f)
		{
			if (medic == null || medic.Dead) return null;
			if (medic.Map == null) return null;

			Map map = medic.Map;

			Pawn best = null;
			float bestD = float.PositiveInfinity;

			IntVec3 center = medic.Position;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(center, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				// Nearby-only scan; no map-wide pawn iteration.
				List<Thing> things = c.GetThingList(map);
				for (int i = 0; i < things.Count; i++)
				{
					Thing t = things[i];
					Pawn p = t as Pawn;
					if (p == null || p.Dead) continue;
					if (p == medic) continue;
					if (p.Faction == null || medic.Faction == null) continue;

					// Must be hostile.
					if (!p.HostileTo(medic)) continue;

					float d = medic.Position.DistanceTo(p.Position);
					if (d > radius) continue;

					if (d < bestD)
					{
						bestD = d;
						best = p;
					}
				}
			}

			return best;
		}

		private IntVec3 FindBestTendSpot(Pawn medic, Pawn patient, float radius = 10f)
		{
			Map map = medic.Map;

			IntVec3 bestCell = medic.Position;
			float bestScore = float.NegativeInfinity;

			bool combatMed = medic.GetComp<CompArgrillianMedicSettings>()?.combatMedic == true;
			Pawn nearestHostile = combatMed ? FindNearestHostile(medic, radius: 80f) : null;
			bool hostilesPresent = nearestHostile != null;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(patient.Position, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;
				if (!c.Standable(map) || !c.Walkable(map)) continue;
				if (c.GetFirstPawn(map) != null && c != medic.Position) continue;

				float d = c.DistanceTo(patient.Position);
				if (d > combatTendMaxDistance) continue;

				if (!medic.CanReach(c, PathEndMode.ClosestTouch, Danger.Some)) continue;

				float score = -d * 3.0f;

				if (hostilesPresent && combatMed)
				{
					bool hasLOS = GenSight.LineOfSight(c, nearestHostile.Position, map);
					if (!hasLOS) score += 40f;
					else score -= 15f;
				}

				if (score > bestScore)
				{
					bestScore = score;
					bestCell = c;
				}
			}

			return bestCell;
		}

		private bool HostilesPresentForMedic(Pawn medic)
		{
			if (medic == null || medic.Map == null) return false;

			Map map = medic.Map;
			float searchRadius = 35f;

			IntVec3 center = medic.Position;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(center, searchRadius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				// Nearby-only: avoid map-wide pawn scans.
				List<Thing> things = c.GetThingList(map);
				for (int i = 0; i < things.Count; i++)
				{
					Thing t = things[i];
					Pawn p = t as Pawn;
					if (p == null || p.Dead) continue;

					if (medic.Faction == null || p.Faction == null) continue;
					if (p.Faction == medic.Faction) continue;
					if (p == medic) continue;

					if (p.HostileTo(medic)) return true;
				}
			}

			return false;
		}

		private static bool IsRescueLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			string n = j.def.defName;

			return j.def == JobDefOf.Rescue || n.IndexOf("rescue", StringComparison.OrdinalIgnoreCase) >= 0;
		}

		private static bool IsMoveLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			string n = j.def.defName.ToLowerInvariant();

			return n.Contains("goto") || n.Contains("move") || n.Contains("walk") || n.Contains("path");
		}

		private static bool IsMealOrConsumeLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;

			// If the job targets medicine, it's not a meal/consume job (even if it uses grab/pickup/haul verbs).
			if (j.targetA.IsValid && j.targetA.Thing != null && IsMedicineThing(j.targetA.Thing)) return false;
			if (j.targetB.IsValid && j.targetB.Thing != null && IsMedicineThing(j.targetB.Thing)) return false;
			if (j.targetC.IsValid && j.targetC.Thing != null && IsMedicineThing(j.targetC.Thing)) return false;

			string n = j.def.defName.ToLowerInvariant();

			return n.Contains("eat")
				|| n.Contains("ingest")
				|| n.Contains("consume")
				|| n.Contains("meal")
				|| n.Contains("cook")
				|| n.Contains("kitchen")
				|| n.Contains("grab")
				|| n.Contains("pickup");
		}

		private static System.Type LastRmType;
		private static System.Reflection.MethodInfo LastWhoReservedMethod;

		private static Pawn GetWhoReserved(Thing thing)
		{
			if (thing == null || thing.Map == null) return null;

			var rm = thing.Map.reservationManager;
			if (rm == null) return null;

			System.Type t = rm.GetType();

			// Micro-cache to avoid repeating reflection when the reservation manager type is constant.
			// (Keeps behavior identical; only reduces overhead.)
			System.Reflection.MethodInfo mi = null;
			if (LastRmType == t)
			{
				mi = LastWhoReservedMethod;
			}
			else
			{
				string[] methodNames =
				{
					"WhoReserved",
					"ReservedBy",
					"ReservedByPawn",
					"GetReserver"
				};

				var methods = t.GetMethods(System.Reflection.BindingFlags.Instance |
				                           System.Reflection.BindingFlags.Public |
				                           System.Reflection.BindingFlags.NonPublic);

				foreach (string name in methodNames)
				{
					foreach (var cand in methods)
					{
						if (cand == null) continue;
						if (!string.Equals(cand.Name, name, System.StringComparison.OrdinalIgnoreCase)) continue;

						if (!typeof(Pawn).IsAssignableFrom(cand.ReturnType)) continue;

						var pars = cand.GetParameters();
						if (pars == null || pars.Length != 1) continue;

						var pType = pars[0].ParameterType;
						if (pType == null) continue;
						if (!pType.IsInstanceOfType(thing)) continue;

						mi = cand;
						break;
					}

					if (mi != null) break;
				}

				LastRmType = t;
				LastWhoReservedMethod = mi;
			}

			if (mi == null) return null;

			try
			{
				object result = mi.Invoke(rm, new object[] { thing });
				var pawn = result as Pawn;

				return pawn;
			}
			catch (System.Exception ex)
			{
				Log.Message($"[JobGiver_TendRetreatingAllies] GetWhoReserved invoke failed: bed={thing} mi={mi.Name} ex={ex.GetType().Name}: {ex.Message}");
				return null;
			}
		}

		private static bool CanReserveThingForPatient(Pawn medic, Thing t, Pawn patient)
		{
			if (medic == null || t == null) return false;
			if (patient == null) return false;
			if (!medic.Spawned || medic.Map == null) return false;
			if (t.Map == null || t.Map != medic.Map) return false;

			if (medic.CanReserve(t, 1, -1, null, false))
				return true;

			Pawn reservedBy = GetWhoReserved(t);
			if (reservedBy != null && reservedBy == patient)
				return true;

			return false;
		}

		private static bool IsCombatAttackLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			string n = j.def.defName;

			return n.IndexOf("attack", StringComparison.OrdinalIgnoreCase) >= 0
				|| n.IndexOf("shoot", StringComparison.OrdinalIgnoreCase) >= 0
				|| n.IndexOf("fight", StringComparison.OrdinalIgnoreCase) >= 0
				|| n.IndexOf("melee", StringComparison.OrdinalIgnoreCase) >= 0
				|| n.IndexOf("ranged", StringComparison.OrdinalIgnoreCase) >= 0
				|| n.IndexOf("burst", StringComparison.OrdinalIgnoreCase) >= 0;
		}

		private static bool MedicJobIsRescueForPatient(Pawn medic, Pawn patient)
		{
			if (medic == null || patient == null) return false;
			if (medic.CurJob == null || medic.CurJob.def == null) return false;

			if (!IsRescueLikeJob(medic.CurJob)) return false;

			// FIX: JobDefOf.Rescue can store the target pawn in different targets (targetA vs targetB/C),
			// and if we check only targetA we may fail to detect “we are already rescuing this patient”.
			Job j = medic.CurJob;

			if (j.targetA.IsValid && j.targetA.Thing == patient) return true;
			if (j.targetB.IsValid && j.targetB.Thing == patient) return true;
			if (j.targetC.IsValid && j.targetC.Thing == patient) return true;

			return false;
		}

		private static readonly System.Collections.Generic.Dictionary<int, Building_Bed> RescueBedCache = new System.Collections.Generic.Dictionary<int, Building_Bed>();

		private static bool IsNonCombatJob(Job j)
		{
			if (j == null || j.def == null) return true;

			if (j.def == JobDefOf.Rescue || j.def == JobDefOf.TendPatient)
				return false;

			return !IsCombatAttackLikeJob(j);
		}

		private static Building_Bed GetCachedRescueBedForPatient(Pawn patient)
		{
			if (patient == null || patient.Map == null) return null;
			if (!RescueBedCache.TryGetValue(patient.thingIDNumber, out var bed) || bed == null) return null;

			if (bed.Destroyed || !bed.Spawned) return null;
			if (bed.Map == null || bed.Map != patient.Map) return null;

			return bed;
		}

		private static void CacheRescueBedForPatient(Pawn patient, Building_Bed bed)
		{
			if (patient == null || bed == null) return;
			RescueBedCache[patient.thingIDNumber] = bed;
		}

		private static void ClearCachedRescueBedForPatient(Pawn patient)
		{
			if (patient == null) return;
			RescueBedCache.Remove(patient.thingIDNumber);
		}

		private static Pawn GetCarriedPawn(Pawn medic)
		{
			if (medic == null) return null;
			if (medic.carryTracker == null) return null;

			return medic.carryTracker.CarriedThing as Pawn;
		}

		private static bool IsMedicCarryingPawn(Pawn medic, Pawn patient)
		{
			if (medic == null || patient == null) return false;

			Pawn carried = GetCarriedPawn(medic);
			return carried != null && carried == patient;
		}

		private static bool IsCrawlLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			return j.def.defName.IndexOf("crawl", StringComparison.OrdinalIgnoreCase) >= 0;
		}

		private bool PatientRecentlyStableForTendOverride(Pawn patient)
		{
			if (patient == null || patient.Dead) return false;

			// Keep this intentionally simple: if they are moving, let combat/tend finish tug-of-war.
			if (patient.pather != null && patient.pather.Moving) return false;

			return true;
		}

		// NEW: keep the patient from starting/continuing other jobs while a combat medic is committing to Rescue/Tend.
		// This prevents "patient keeps attacking/hauling/moving" so the medic can actually complete tending/rescue.
		// -------------------- PATCH 1: LockPatientToMedic --------------------
		// This prevents the "wiggle/standing/hauling loop" without breaking the tend/rescue pipeline.
		// This prevents "patient keeps attacking/hauling/moving" so the medic can actually complete tending/rescue.
		// Non-destructive: do NOT StopAll(true) and do NOT inject Wait jobs (they destabilize job arbitration during the transition).
		private static void LockPatientToMedic(Pawn medic, Pawn patient)
		{
			if (patient == null || patient.Dead) return;

			ArgrillianMedicalState.PatientMedicHold.Lock(medic, patient);

			// Do NOT stop the patient here.
			// The patient must keep retreating / repositioning toward safety and/or toward the medic,
			// so that the tend job can actually succeed when eligibility is reached.

			// Do NOT interrupt/clear the patient’s current job here either.
			// Interfering too early commonly causes “stuck tending” (medic starts tend but patient never reaches the correct tend state).
			//
			// We only hard-stop right before we start Tend/Rescue (inside the heldPatient + canTendNow branch).
		}

		protected override Job TryGiveJob(Pawn pawn)
		{
			if (pawn == null || pawn.Map == null) return null;

			var medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
			if (medicComp == null || !medicComp.isMedic) return null;
			if (!medicComp.combatMedic) return null;

			Pawn heldPatient = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);
			if (heldPatient != null)
			{
				// If we are already on Tend/Rescue but tend is no longer valid/eligible,
				// cancel the medical job so the medic can re-attempt later.
				if (pawn.CurJob != null && pawn.CurJob.def != null)
				{
					if (pawn.CurJob.def == JobDefOf.TendPatient)
					{
						if (!canTendNow(pawn, heldPatient))
							pawn.jobs.EndCurrentJob(JobCondition.InterruptForced);
						else
							return pawn.CurJob;
					}
					else if (pawn.CurJob.def == JobDefOf.Rescue)
					{
						if (!heldPatient.Downed)
							pawn.jobs.EndCurrentJob(JobCondition.InterruptForced);
						else
							return pawn.CurJob;
					}
				}

				bool medicIsActiveMedical =
					pawn.CurJob != null &&
					pawn.CurJob.def != null &&
					(pawn.CurJob.def == JobDefOf.TendPatient || pawn.CurJob.def == JobDefOf.Rescue);

				// When we are in/approaching Tend/Rescue, prevent the patient from continuing
				// their combat job while we’re able to start tending (fixes "medic stuck +
				// patient keeps fighting").
				bool canTouchNow = canTendNow(pawn, heldPatient);

				if (medicIsActiveMedical)
				{
					// Stop movement drift + prevent job-queue switching.
					heldPatient.pather?.StopDead();
					heldPatient.jobs?.ClearQueuedJobs();

					// IMPORTANT: we do NOT force-start an attack job here.
					// If an enemy is already in range, RimWorld’s attack/shoot behavior will execute
					// without needing chase/move.

					bool isHeldPatientOnCombatJob =
						heldPatient.CurJob != null &&
						heldPatient.CurJob.def != null &&
						(
							heldPatient.CurJob.def == JobDefOf.AttackMelee ||
							heldPatient.CurJob.def == JobDefOf.AttackStatic ||
							(heldPatient.CurJob.def.defName != null && (
								heldPatient.CurJob.def.defName.IndexOf("attack", System.StringComparison.OrdinalIgnoreCase) >= 0 ||
								heldPatient.CurJob.def.defName.IndexOf("shoot", System.StringComparison.OrdinalIgnoreCase) >= 0 ||
								heldPatient.CurJob.def.defName.IndexOf("range", System.StringComparison.OrdinalIgnoreCase) >= 0
							))
						);

					// If the patient is in a combat job, only interrupt when they would need to
					// move (i.e., their current hostile target is NOT in range).
					// This keeps “shooting/attacking in-range” allowed while still preventing movement derailment.
					if (isHeldPatientOnCombatJob)
					{
						// Try to infer current hostile target from the active job.
						LocalTargetInfo jobTarget =
							(heldPatient.CurJob.targetA.IsValid) ? heldPatient.CurJob.targetA :
							(heldPatient.CurJob.targetB.IsValid) ? heldPatient.CurJob.targetB :
							(LocalTargetInfo) LocalTargetInfo.Invalid;

						if (jobTarget.IsValid && jobTarget.Thing != null)
						{
							float distSqr = jobTarget.Thing != null ? heldPatient.Position.DistanceToSquared(jobTarget.Thing.Position) : heldPatient.Position.DistanceToSquared(jobTarget.Cell);

							// Approximate reach: melee uses adjacency; ranged uses current attack verb range when possible.
							float maxDistSqr = 0.0f;

							if (heldPatient.CurJob.def == JobDefOf.AttackMelee)
							{
								maxDistSqr = 2.25f; // ~1.5 tiles; melee-ish
							}
							else
							{
								Verb attackVerb = null;
								if (jobTarget.Thing != null)
								{
									// RimWorld 1.6 signature requires: TryGetAttackVerb(Thing target, bool allowIndirectFire, bool allowMelee)
									// Using (true, true) lets the verb resolve for both melee and ranged weapons.
									attackVerb = heldPatient.TryGetAttackVerb(jobTarget.Thing, true, true);
								}

								float reach = (attackVerb != null && attackVerb.verbProps != null) ? attackVerb.verbProps.range : 0f;

								// If we can't compute reach, be conservative and prevent movement derailment.
								if (reach > 0f)
									maxDistSqr = (reach * reach) + 0.25f;
								else
									maxDistSqr = 0.0f;
							}

							bool targetInRangeNow = (maxDistSqr > 0.0f) ? (distSqr <= maxDistSqr) : false;

							// Rule: patient may attack/shoot while in range, but MUST NOT move (no chasing/advancing).
							bool patientIsMoving = heldPatient.pather != null && heldPatient.pather.Moving;

							if (!targetInRangeNow)
							{
								// Not in range -> stop advancing/chasing so tend can proceed.
								heldPatient.jobs.EndCurrentJob(JobCondition.InterruptForced);
							}
							else
							{
								// In range -> allow combat, but remove drift/pathing that would derail tending.
								// Stop movement and clear queued jobs only; do NOT EndCurrentJob for combat.
								if (patientIsMoving)
								{
									heldPatient.pather?.StopDead();
									heldPatient.jobs?.ClearQueuedJobs();
								}
							}
						}
						else
						{
							// No valid target info -> safest behavior: interrupt so patient can't wander.
							heldPatient.jobs.EndCurrentJob(JobCondition.InterruptForced);
						}
					}
					else
					{
						// Non-combat (haul/wait/flee/etc.) should be interrupted immediately to keep tend exclusive.
						heldPatient.jobs.EndCurrentJob(JobCondition.InterruptForced);
					}

					// IMPORTANT: we do NOT force-start an attack job here.
					// If an enemy is already in range, RimWorld’s attack/shoot behavior will execute
					// without needing chase/move.
				}

				// Tend-first arbitration:
				// If the currently held patient is NOT the most serious eligible target,
				// release and re-lock onto the best patient. This prevents “held patient not actually
				// top-priority -> medic drifts to next jobs / battles” behavior.
				if (!canTendNow(pawn, heldPatient))
				{
					Pawn bestCandidate = ArgrillianAlertSystem.GetBestPatientFromCalls(pawn, searchRadius);

					if (bestCandidate != null && bestCandidate != heldPatient)
					{
						// Cooldown: prevent rapid release/re-lock while still not eligible to tend.
						// This keeps progress monotonic and avoids thrash when PatientCalls update.
						if (!RecentlySwitchedTendLock(pawn, bestCandidate, minJobSwitchCooldownTicks))
						{
							// Prefer downed over merely-injured.
							if (bestCandidate.Downed && !heldPatient.Downed)
							{
								ArgrillianMedicalState.PatientMedicHold.ReleaseForMedic(pawn);
								ArgrillianMedicalState.PatientMedicHold.Lock(pawn, bestCandidate);
								heldPatient = bestCandidate;
								MarkSwitchedTendLock(pawn, heldPatient);
							}
							else
							{
								// Otherwise prefer lower HP% (more serious) if both are eligible candidates.
								float heldHpPct = heldPatient.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
								float candHpPct = bestCandidate.health?.summaryHealth?.SummaryHealthPercent ?? 1f;

								if (candHpPct < heldHpPct)
								{
									ArgrillianMedicalState.PatientMedicHold.ReleaseForMedic(pawn);
									ArgrillianMedicalState.PatientMedicHold.Lock(pawn, bestCandidate);
									heldPatient = bestCandidate;
									MarkSwitchedTendLock(pawn, heldPatient);
								}
							}
						}
					}
				}

				// After lock arbitration, attempt Tend/Rescue.
				if (canTendNow(pawn, heldPatient))
				{
					// Only at tend-start: hard-stop patient drift so tend/rescue completion isn't blocked.
					heldPatient.pather?.StopDead();
					return JobMaker.MakeJob(JobDefOf.TendPatient, heldPatient);
				}

				// If downed and a bed exists, try Rescue.
				if (heldPatient.Downed)
				{
					Building_Bed bed;
					if (TryGetRescueBedForPatient(pawn, heldPatient, out bed))
					{
						heldPatient.pather?.StopDead();
						return JobMaker.MakeJob(JobDefOf.Rescue, heldPatient, bed);
					}
				}

				// Not eligible yet:
				// Keep the medic in the medical pipeline without tend/rescue,
				// but DO NOT just Wait.
				IntVec3 tendSpot = FindBestTendSpot(pawn, heldPatient, combatTendMaxDistance);

				// Fallback: avoid feeding an invalid/unreachable cell into Goto (causes idle/standing).
				if (!tendSpot.IsValid)
				{
					tendSpot = pawn.Position;
				}

				return ArgrillianGotoHelper.MakeGotoWithNoChurn(pawn, tendSpot);
			}

			return JobMaker.MakeJob(JobDefOf.Wait, 60);
		}

		private bool TryGetRescueBedForPatient(Pawn medic, Pawn patient, out Building_Bed bed)
		{
			bed = null;
			if (medic == null || patient == null)
				return false;
			if (medic.Map == null || patient.Map == null)
				return false;
			if (medic.Map != patient.Map)
				return false;
			if (patient.Dead || medic.Dead)
				return false;
			if (!medic.Spawned || !patient.Spawned)
				return false;

			Map map = medic.Map;

			// Prefer cached bed if still valid.
			Building_Bed cached = GetCachedRescueBedForPatient(patient);
			if (cached != null)
			{
				bed = cached;
				return CanReserveThingForPatient(medic, cached, patient);
			}

			Building_Bed best = null;
			float bestDist = float.PositiveInfinity;

			foreach (var b in map.listerBuildings.AllBuildingsColonistOfClass<Building_Bed>())
			{
				if (b == null) continue;
				if (b.Destroyed || !b.Spawned) continue;
				if (b.Map != map) continue;

				float dist = b.Position.DistanceToSquared(patient.Position);
				if (dist > (hospitalBedMaxDist * hospitalBedMaxDist))
					continue;

				if (!CanReserveThingForPatient(medic, b, patient))
					continue;

				if (dist < bestDist)
				{
					bestDist = dist;
					best = b;
				}
			}

			if (best == null)
				return false;

			CacheRescueBedForPatient(patient, best);
			bed = best;
			return true;
		}

		private bool canTendNow(Pawn pawn, Pawn patient)
		{
			bool combatMedic = pawn.GetComp<CompArgrillianMedicSettings>()?.combatMedic == true;

			// Held patient invariant: if this patient is locked into this medic's medical pipeline,
			// tend/rescue eligibility should NOT be blocked by the patient's current combat job.
			if (combatMedic)
			{
				Pawn heldPatient = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);

				bool holdActive =
					heldPatient != null &&
					patient != null &&
					heldPatient.thingIDNumber == patient.thingIDNumber;

				if (holdActive)
				{
					return IsValidTendTarget(patient, pawn) &&
						CanReserveTendTarget(pawn, patient);
				}
			}

			// Normal (not-held) case keeps the stricter checks you already had.
			int stableTicksOuter = GetPatientStableTicksForTend(patient);
			int requiredStableTicksOuter = patient.Downed ? 0 : patientStableTicksRequired;

			if (!patient.Downed && patient.CurJob != null && patient.CurJob.def != null)
			{
				Job j = patient.CurJob;

				// Don’t start Tend if the patient is actively doing combat/mobility jobs (only in normal case).
				if (IsCombatAttackLikeJob(j) || IsChaseOrTacticJob(j) || IsFleeLikeJob(j) || IsHaulJob(j) || IsMealOrConsumeLikeJob(j) || j.def.defName == "ConsumeMeal")
				{
					return false;
				}

				if (j.def == JobDefOf.LayDown)
				{
					return false;
				}

				if (j.def == JobDefOf.Rescue)
				{
					return false;
				}

				if (!IsAllowedDownPawnJob(j))
				{
					return false;
				}
			}

			bool distanceOk = patient.Downed ? true : pawn.Position.DistanceTo(patient.Position) <= combatTendMaxDistance;

			return distanceOk &&
				IsValidTendTarget(patient, pawn) &&
				stableTicksOuter >= requiredStableTicksOuter &&
				CanReserveTendTarget(pawn, patient);
		}

		private struct PatientStabilityState
		{
			public IntVec3 lastPos;
			public int stableTicks;
			public int lastSeenTick;
		}

		private static readonly System.Collections.Generic.Dictionary<int, PatientStabilityState> PatientStabilityCache = new System.Collections.Generic.Dictionary<int, PatientStabilityState>();

		private static int GetPatientStableTicksForTend(Pawn patient)
		{
			if (patient == null) return 0;
			if (patient.Dead) return 0;
			if (patient.Map == null) return 0;

			int id = patient.thingIDNumber;
			int tick = Find.TickManager.TicksGame;
			IntVec3 pos = patient.Position;

			if (!PatientStabilityCache.TryGetValue(id, out var st))
			{
				st = new PatientStabilityState
				{
					lastPos = pos,
					stableTicks = 0,
					lastSeenTick = tick
				};
				PatientStabilityCache[id] = st;
				return 0;
			}

			int dt = tick - st.lastSeenTick;
			if (dt < 0) dt = 0;

			if (pos == st.lastPos)
				st.stableTicks = st.stableTicks + dt;
			else
			{
				st.lastPos = pos;
				st.stableTicks = 0;
			}

			st.lastSeenTick = tick;

			// Prevent runaway values from extreme tick gaps.
			if (st.stableTicks > 200000) st.stableTicks = 200000;

			PatientStabilityCache[id] = st;
			return st.stableTicks;
		}

		private static bool IsFleeLikeJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;

			string n = j.def.defName.ToLowerInvariant();

			return n.Contains("flee")
				|| n.Contains("runaway")
				|| n.Contains("runfrom")
				|| n.Contains("escape")
				|| n.Contains("evade")
				|| n.Contains("retreat")
				|| n.Contains("withdraw");
		}

		private static bool IsChaseOrTacticJob(Job j)
		{
			if (j == null || j.def == null || string.IsNullOrEmpty(j.def.defName)) return false;
			string n = j.def.defName.ToLowerInvariant();

			return n.Contains("chase")
				|| n.Contains("pursuit")
				|| n.Contains("hunt")
				|| n.Contains("stalk")
				|| n.Contains("tactic")
				|| n.Contains("seek");
		}
	}
}