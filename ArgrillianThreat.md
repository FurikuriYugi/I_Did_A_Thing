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

			return parent.Map.listerThings.AllThings
				.OfType<Pawn>()
				.FirstOrDefault(p => p.thingIDNumber == thingID && !p.Dead);
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

			// If the medic calling this helper is already tending/rescuing this same patient,
			// do not yield/abort due to other pawns also targeting the patient this tick.
			if (medic.CurJob != null && JobIsMedicalForPatient(medic.CurJob, patient)) return null;

			Map map = patient.Map;

			// Performance: do NOT scan AllPawnsSpawned.
			// Nearby-only: enough to answer "is anyone already doing the Tend/Rescue for this patient right now?"
			// Keep radius small to avoid long-range / whole-map work.
			float searchRadius = 12f;

			IntVec3 center = patient.Position;

			float best = float.PositiveInfinity;

			// Iterate things in nearby cells and only consider pawns from those cells.
			// (Much smaller set than map.mapPawns.AllPawnsSpawned.)
			foreach (IntVec3 c in GenRadial.RadialCellsAround(center, searchRadius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				float d = c.DistanceTo(center);
				if (d > best) continue;
				// best is not strictly required here; leaving it as a tiny pruning knob.

				foreach (Thing t in c.GetThingList(map))
				{
					Pawn p = t as Pawn;
					if (p == null) continue;
					if (p == medic) continue;
					if (p.Dead) continue;
					if (!p.Spawned || p.Map != map) continue;
					if (p.jobs == null || p.jobs.curJob == null) continue;

					Job cur = p.jobs.curJob;
					if (cur == null || cur.def == null) continue;

					bool isMedical =
						cur.def == JobDefOf.TendPatient ||
						cur.def == JobDefOf.Rescue;

					if (!isMedical)
						continue;

					if (!JobTargetsIncludePawn(cur, patient))
						continue;

					// Return first qualifying pawn; nearby list keeps this cheap.
					return p;
				}
			}

			return null;
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
					lastSeenTick = now
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

				Pawn p = null;
				foreach (Pawn candidate in pursuer.Map.mapPawns.AllPawnsSpawned)
				{
					if (candidate != null && candidate.thingIDNumber == locked.hostileId)
					{
						p = candidate;
						break;
					}
				}

				if (p == null) return null;
				if (p.Dead) return null;
				if (!p.Spawned) return null;
				if (pursuer.Map == null || p.Map != pursuer.Map) return null;
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
			private static readonly Dictionary<int, int> medicByPatient = new Dictionary<int, int>();
			private static readonly Dictionary<int, int> patientByMedic = new Dictionary<int, int>();

			public static Pawn GetHeldPatient(Pawn medic)
			{
				if (medic == null) return null;

				int medicId = medic.thingIDNumber;
				if (!patientByMedic.TryGetValue(medicId, out int patientId)) return null;

				return FindPatientByThingID(medic.Map, patientId);
			}

			

			public static void Lock(Pawn medic, Pawn patient)
			{
				if (medic == null || patient == null || patient.Dead) return;

				int medicId = medic.thingIDNumber;
				int patientId = patient.thingIDNumber;

				patientByMedic[medicId] = patientId;
				medicByPatient[patientId] = medicId;
			}

			public static void ReleaseForMedic(Pawn medic)
			{
				if (medic == null) return;

				int medicId = medic.thingIDNumber;
				if (!patientByMedic.TryGetValue(medicId, out int patientId)) return;

				patientByMedic.Remove(medicId);
				medicByPatient.Remove(patientId);

				Pawn patient = FindPatientByThingID(medic.Map, patientId);
				if (patient == null || patient.Dead) return;

				// CRITICAL: do NOT ClearQueuedJobs() or StopAll here.
				// Clearing queued jobs during tend/rescue transition can leave the patient in a half-state
				// (getting up / stuck moving), and then the normal utility layer can steer into hauling.
				patient.pather?.StopDead();
			}

			private static Pawn FindPatientByThingID(Map map, int thingID)
			{
				if (map == null) return null;

				foreach (Thing t in map.spawnedThings)
				{
					if (t != null && t.thingIDNumber == thingID)
						return t as Pawn;
				}

				return null;
			}
		}
	}

	public static class ArgrillianThreatAlert
	{
		public static void BroadcastSharedAwareness(
			Pawn source,
			Pawn hostile,
			float allyRadius,
			bool squadOnly)
		{
			if (source == null || hostile == null) return;
			if (source.Map == null) return;
			if (hostile.Dead || !hostile.Spawned) return;

			Map map = source.Map;
			IntVec3 lastKnown = hostile.Position;
			int hostileId = hostile.thingIDNumber;

			// Use killed/hostile position as the "last known"
			var allies = map.mapPawns.AllPawnsSpawned;
			foreach (Pawn p in allies)
			{
				if (p == null || p.Dead || !p.Spawned) continue;
				if (p == source) continue;
				if (p.Faction != source.Faction) continue;
				if (p.Map != map) continue;

				if (p.Position.DistanceTo(source.Position) > allyRadius) continue;

				if (squadOnly)
				{
					var ts = p.GetComp<CompArgrillianThreatSettings>();
					if (ts == null || ts.squadMode != true) continue;
				}

				// Do not overwrite if they already have fresher/high-alert data:
				// (optional; keeps behavior from "chasing stale alerts")
				if (ArgrillianThreatState.AwarenessCache.IsHighAlert(p) || ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(p))
					continue;

				ArgrillianThreatState.AwarenessCache.MarkShared(p, lastKnown, hostileId);
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

	// -----------------------------------------
	// Note: Check why there is only IsRanged in here and not melee and what this actually does
	// -----------------------------------------
	public readonly struct HostileContext
	{
		public Pawn Hostile { get; init; }
		public Verb AttackVerb { get; init; }
		public bool IsRanged { get; init; }
	}

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

			Map map = patient.Map;

			foreach (Pawn p in map.mapPawns.AllPawnsSpawned)
			{
				if (p == null || p.Dead || !p.Spawned) continue;

				var comp = p.GetComp<CompArgrillianMedicSettings>();
				if (comp == null || !comp.isMedic) continue;

				Job cur = p.CurJob;
				if (cur == null) continue;

				if (cur.def == JobDefOf.TendPatient || cur.def == JobDefOf.Rescue)
				{
					// TendPatient uses targetA as the patient pawn (vanilla style).
					if (cur.targetA.Thing as Pawn == patient)
						return true;
				}
			}

			return false;
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
				med = FindNearestMedic(pawn, map, patientRetreatPreferMedicRadius, requireCombatMedicOrEither: false);

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
						// Condition: still in the "being tended by Argrillian medic" relationship.
						bool patientStillBoundToMedic = IsBeingTendedByArgrillianMedic(patient);

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

		private static Pawn FindNearestMedic(Pawn patient, Map map, float radius, bool requireCombatMedicOrEither)
		{
			if (patient == null || map == null) return null;

			Pawn best = null;
			float bestDist = float.PositiveInfinity;

			foreach (IntVec3 c in GenRadial.RadialCellsAround(patient.Position, radius, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn p) continue;
					if (p.Dead || !p.Spawned || p.Map != map) continue;
					if (p.Faction != patient.Faction) continue;

					var ms = p.GetComp<CompArgrillianMedicSettings>();
					if (ms == null || !ms.isMedic) continue;

					if (requireCombatMedicOrEither && !ms.combatMedic)
						continue;

					float d = p.Position.DistanceTo(patient.Position);
					if (d < bestDist && p.CanReach(patient.Position, PathEndMode.ClosestTouch, Danger.None))
					{
						bestDist = d;
						best = p;
					}
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

		public Pawn AssignedPawn
		{
			get => ArgrillianGizmoHelpers.FindAlivePawnByThingID(parent, assignedPawnThingID);
		}

		public override IEnumerable<Gizmo> CompGetGizmosExtra()
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
					if (!isMedic)
					{
						combatMedic = false;
						assignedPawnThingID = -1;
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
					if (combatMedic) isMedic = true;
				}
			);
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
		private float combatMedicAidHPPercentThreshold = 0.75f;        // ally injured enough to trigger medic aid
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

		private static class NearestMedicCache
		{
			private struct CacheEntry
			{
				public int tick;
				public int mapId;
				public Pawn medic;
			}

			private static readonly Dictionary<int, CacheEntry> bySeekerId
				= new Dictionary<int, CacheEntry>();

			public static Pawn GetOrCompute(Pawn seeker, Func<Pawn> compute)
			{
				if (seeker == null || seeker.Dead || seeker.Map == null) return null;

				int now = Find.TickManager.TicksGame;
				int seekerId = seeker.thingIDNumber;
				int mapId = seeker.Map.uniqueID;

				if (bySeekerId.TryGetValue(seekerId, out CacheEntry e))
				{
					if (e.tick == now && e.mapId == mapId)
						return e.medic;
				}

				Pawn result = compute();

				bySeekerId[seekerId] = new CacheEntry
				{
					tick = now,
					mapId = mapId,
					medic = result
				};

				return result;
			}
		}

		private Pawn FindNearestMedic(Pawn seeker)
		{
			return NearestMedicCache.GetOrCompute(seeker, () =>
			{
				if (seeker == null || seeker.Dead || seeker.Map == null) return null;

				Map map = seeker.Map;
				Pawn best = null;
				float bestD = float.PositiveInfinity;

				foreach (Pawn p in map.mapPawns.AllPawnsSpawned)
				{
					if (p == null || p.Dead) continue;
					if (p.Faction != seeker.Faction) continue;
					if (!p.Spawned || p.Map != map) continue;
					if (p.Downed) continue;

					var medicComp = p.GetComp<CompArgrillianMedicSettings>();
					if (medicComp == null || !medicComp.isMedic) continue;

					float d = seeker.Position.DistanceTo(p.Position);
					if (d < bestD)
					{
						bestD = d;
						best = p;
					}
				}

				return best;
			});
		}

		private Pawn FindBestAidTargetForCombatMedic(Pawn medic, float maxRange)
		{
			if (medic == null || medic.Dead || medic.Map == null) return null;

			var medicComp = medic.GetComp<CompArgrillianMedicSettings>();
			Pawn assignedPatient = medicComp?.AssignedPawn;

			// If the assigned patient is already being tended/rescued by someone else,
			// don't let this combat medic start competing.
			if (assignedPatient != null)
			{
				if (assignedPatient.Spawned && assignedPatient.Map == medic.Map && !assignedPatient.Dead)
				{
					// If we're already on a medical job for the assigned patient, keep going.
					if (medic.CurJob != null && medic.CurJob.def != null)
					{
						Pawn curPatient = ArgillianThreatPatientTuning.GetPatientFromJob(medic.CurJob);
						if ((medic.CurJob.def == JobDefOf.Rescue || medic.CurJob.def == JobDefOf.TendPatient) && curPatient == assignedPatient)
							return assignedPatient;
					}

					Pawn someoneElse = ArgillianThreatPatientTuning.PatientAlreadyBeingTendedOrRescuedBySomeoneElse(medic, assignedPatient);
					if (someoneElse != null)
						return null;

					// Keep assisting until the patient is already being tended/rescued.
					// This prevents the medic from “dropping” the assist just because HP% rises above thresholds.
					float dAssigned = medic.Position.DistanceTo(assignedPatient.Position);
					if (dAssigned <= maxRange)
						return assignedPatient;
				}
			}

			float bestScore = float.NegativeInfinity;
			Pawn best = null;

			foreach (Pawn p in medic.Map.mapPawns.AllPawnsSpawned)
			{
				if (p == null || p.Dead) continue;
				if (!p.Spawned || p.Map != medic.Map) continue;
				if (p.Faction != medic.Faction) continue;

				float hpPct = p.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
				bool downed = p.Downed;

				// If not downed, it must be injured enough.
				if (!downed && hpPct >= combatMedicAidHPPercentThreshold)
					continue;

				float d = medic.Position.DistanceTo(p.Position);
				if (d > maxRange)
					continue;

				// Avoid picking someone already being tended/rescued by someone else.
				// FIX: if the target is downed, we must be willing to take it even if
				// the "already being tended/rescued" predicate is overly strict,
				// otherwise medics will never enter tend/rescue and will fall through to
				// fight -> meal -> haul behavior.
				if (!downed)
				{
					if (ArgillianThreatPatientTuning.PatientAlreadyBeingTendedOrRescuedBySomeoneElse(medic, p) != null)
						continue;
				}

				float score =
					1000f * (downed ? 1f : 0.25f) +
					(1f - hpPct) * 500f -
					d * 2f;

				if (score > bestScore)
				{
					bestScore = score;
					best = p;
				}
			}

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
		private static readonly Dictionary<(int attackerId, int hostileId, int mapId), HostileMotionSample> HostileMotion =
			new Dictionary<(int, int, int), HostileMotionSample>();

		private static int HostileMotionMaxEntries = 4096;
		private static int HostileMotionEntryTtlTicks = 900;

		private static HostileMotionSample UpdateAndGetMotion(Pawn attacker, Pawn hostile)
		{
			if (attacker == null || hostile == null || attacker.Map == null || hostile.Map == null)
				return default;

			int tick = Find.TickManager?.TicksGame ?? 0;

			// Very lightweight pruning to avoid unbounded dictionary growth.
			if (HostileMotion.Count > HostileMotionMaxEntries)
			{
				HostileMotion.Clear();
			}
			else if (HostileMotion.Count > (HostileMotionMaxEntries * 0.75f))
			{
				// TTL-prune occasionally when near the cap.
				var removeKeys = new List<(int, int, int)>();
				foreach (var kvp in HostileMotion)
				{
					if (!kvp.Value.initialized) continue;
					if (tick - kvp.Value.lastTick > HostileMotionEntryTtlTicks)
						removeKeys.Add(kvp.Key);
				}
				for (int i = 0; i < removeKeys.Count; i++)
					HostileMotion.Remove(removeKeys[i]);
			}

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

		private Pawn FindAssignedCombatMedicForPatient(Pawn patient)
		{
			if (patient == null || patient.Dead || patient.Map == null) return null;

			var map = patient.Map;

			foreach (Pawn p in map.mapPawns.AllPawnsSpawned)
			{
				if (p == null || p.Dead) continue;
				if (!p.Spawned || p.Map != map) continue;
				if (patient.Faction == null || p.Faction == null) continue;
				if (p.Faction != patient.Faction) continue;

				var medicComp = p.GetComp<CompArgrillianMedicSettings>();
				if (medicComp == null || !medicComp.isMedic || !medicComp.combatMedic) continue;

				if (medicComp.AssignedPawn == patient)
					return p;
			}

			return null;
		}

		private bool PatientRecentlyStableForTendOverride(Pawn patient)
		{
			if (patient == null || patient.Dead) return false;

			// Keep this intentionally simple: if they are moving, let combat/tend finish tug-of-war.
			if (patient.pather != null && patient.pather.Moving) return false;

			return true;
		}

		protected override Job TryGiveJob(Pawn pawn)
		{
			// Medic gating: non-combat medics don't do threat response.
			var medicThreatSettings = pawn?.GetComp<CompArgrillianMedicSettings>();
			if (medicThreatSettings != null && medicThreatSettings.isMedic && !medicThreatSettings.combatMedic)
			{
				return null;
			}

			if (pawn == null || pawn.Dead || pawn.Map == null) return null;
			if (pawn.Downed) return null;

			var map = pawn.Map;

			// TEND OVERRIDE
			Pawn assignedMedic = FindAssignedCombatMedicForPatient(pawn);
			if (assignedMedic != null)
			{
				if (PatientRecentlyStableForTendOverride(pawn))
				{
					return null; // let medic AI and Tend/Rescue jobs control this pawn
				}
			}

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

			ArgrillianThreatState.AwarenessCache.MarkDirect(pawn, hostile);

			ArgrillianThreatAlert.BroadcastSharedAwareness(
				source: pawn,
				hostile: hostile,
				allyRadius: 30f,
				squadOnly: true
			);

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
					Pawn nearestMedic = FindNearestMedic(pawn);

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
							Pawn nearestMedic = FindNearestMedic(pawn);

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
							Pawn nearestMedic = FindNearestMedic(pawn);

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
				Pawn nearestMedic = FindNearestMedic(pawn);

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

			bool alerted =
				ArgrillianThreatState.AwarenessCache.IsHighAlert(pawn) ||
				ArgrillianThreatState.AwarenessCache.IsSharedInvestigate(pawn);

			if (alerted && !skipDecisionTick)
			{
				bool hasLOS = (hostile != null && !hostile.Dead && hostile.Spawned && hostile.Map == map)
					&& GenSight.LineOfSight(pawn.Position, hostile.Position, map);

				if (hasLOS)
				{
					bool immediateThreat = IsImmediateThreat(pawn, hostile, desiredCombatDistanceNow);

					bool useImmediateHardGate = isCombatMedic || shouldRetreat || wantsPatientRetreat;

					if (!immediateThreat && useImmediateHardGate)
					{
						Pawn nearestMedic = FindNearestMedic(pawn);
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

		private Pawn FindBestPatient(Pawn medic)
		{
			Map map = medic.Map;
			Pawn best = null;
			float bestScore = float.NegativeInfinity;

			float r = ArgrillianThreatMath.ClampRadialRadius(searchRadius);

			foreach (IntVec3 c in GenRadial.RadialCellsAround(medic.Position, r, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn p) continue;
					if (p == medic) continue;
					if (p.Dead || p.Faction != medic.Faction) continue;

					float hpPct = p.health.summaryHealth.SummaryHealthPercent;
					if (hpPct > 0.95f) continue;

					bool bleeding = HasBloodLossStatic(p);
					bool serious = hpPct <= treatBelowHealthPercent;

					if (!bleeding && !serious && !p.Downed) continue;

					float score = PatientUrgencyStatic(p, treatBelowHealthPercent) - medic.Position.DistanceTo(p.Position) * 0.35f;

					if (p.Downed)
						score += downedPatientScoreBonus;

					Pawn nearestHostile = FindNearestHostile(medic, radius: 80f);
					if (nearestHostile != null)
					{
						float dToHostile = p.Position.DistanceTo(nearestHostile.Position);
						if (dToHostile < 18f) score -= 40f;
					}

					if (score > bestScore)
					{
						bestScore = score;
						best = p;
					}
				}
			}

			return best;
		}

		private Pawn FindNearestHostile(Pawn medic, float radius)
		{
			Map map = medic.Map;

			bool finishOff = false;
			{
				CompArgrillianThreatSettings s = medic.GetComp<CompArgrillianThreatSettings>();
				finishOff = s != null && s.finishOff;
			}

			Pawn best = null;
			float bestDist = float.PositiveInfinity;

			float r = ArgrillianThreatMath.ClampRadialRadius(radius);

			foreach (IntVec3 c in GenRadial.RadialCellsAround(medic.Position, r, true))
			{
				if (!c.InBounds(map) || c.Fogged(map)) continue;

				foreach (Thing t in c.GetThingList(map))
				{
					if (t is not Pawn p) continue;
					if (p.Dead) continue;
					if (p.Faction == medic.Faction) continue;

					// NEW: When Finish Off is false, downed hostiles must be ignored as hostile anchors
					// so melee doesn't hover/reposition around them.
					if (p.Downed && !finishOff) continue;

					float d = medic.Position.DistanceTo(p.Position);
					if (d < bestDist)
					{
						bestDist = d;
						best = p;
					}
				}
			}

			return best;
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
			if (medic?.Map == null) return false;

			foreach (Pawn p in medic.Map.mapPawns.AllPawnsSpawned)
			{
				if (p == null || p.Dead) continue;
				if (p.Faction == null || medic.Faction == null) continue;
				if (p.Faction == medic.Faction) continue;

				if (p.HostileTo(medic)) return true;
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

		// NEW: keep the patient from starting/continuing other jobs while a combat medic is committing to Rescue/Tend.
		// This prevents "patient keeps attacking/hauling/moving" so the medic can actually complete tending/rescue.
		// -------------------- PATCH 1: LockPatientToMedic --------------------
		// This prevents the "wiggle/standing/hauling loop" without breaking the tend/rescue pipeline.
		// This prevents "patient keeps attacking/hauling/moving" so the medic can actually complete tending/rescue.
		// Non-destructive: do NOT StopAll(true) and do NOT inject Wait jobs (they destabilize job arbitration during the transition).
		private static void LockPatientToMedic(Pawn medic, Pawn patient)
		{
			if (patient == null || patient.Dead) return;

			// Record the hold so we can release it when the medic's Tend/Rescue ends.
			ArgrillianMedicalState.PatientMedicHold.Lock(medic, patient);

			// Stop movement so they don't keep trying to flee/turn/position mid-tend.
			patient.pather?.StopDead();

			// Intentionally no:
			// - patient.jobs?.StopAll(true)
			// - injected JobDefOf.Wait / queue-fronting
			//
			// The medic job giver should be the authority that selects Tend/Rescue when eligible.
		}

		// (Existing class continues below)
		// This method is broken and needs to be recreated it is left here for refrence only
		protected override Job TryGiveJob(Pawn pawn)
		{
			if (pawn == null || pawn.Dead || pawn.Map == null) return null;

			var medicComp = pawn.GetComp<CompArgrillianMedicSettings>();
			if (medicComp == null || !medicComp.isMedic) return null;

			bool combatMedic = medicComp.combatMedic;

			// RELEASE FIX (stability): only release the hold when we are truly leaving the medic's assigned medical pipeline.
			// Anti-preemption: medicine-fetch / haul must NOT release Tend/Rescue stickiness while the medic is committed.
			if (combatMedic)
			{
				Pawn heldPatient = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);

				if (heldPatient != null)
				{
					// Hard invariant for combat medics:
					// If we have a held pipeline patient, we MUST not fall through to vanilla “normal jobs”.
					// We also MUST not release the hold inside this branch.
					// Either we keep/continue Tend/Rescue, or we return a pinned medical Wait until canTendNow flips.

					// 1) If we’re already on Tend/Rescue targeting the held patient, keep that job.
					if (pawn.CurJob != null && pawn.CurJob.def != null)
					{
						if (pawn.CurJob.def == JobDefOf.TendPatient || pawn.CurJob.def == JobDefOf.Rescue)
						{
							// Tend/Rescue should target the held pawn.
							if (pawn.CurJob.targetA.IsValid && pawn.CurJob.targetA.Thing == heldPatient)
							{
								return pawn.CurJob;
							}
						}

						// Also protect against approach/move variants that still “belong” to this patient pipeline.
						bool targetsHeldPatient =
							(pawn.CurJob.targetA.IsValid && pawn.CurJob.targetA.Thing == heldPatient) ||
							(pawn.CurJob.targetB.IsValid && pawn.CurJob.targetB.Thing == heldPatient) ||
							(pawn.CurJob.targetC.IsValid && pawn.CurJob.targetC.Thing == heldPatient);

						if (targetsHeldPatient)
						{
							// Keep current job rather than releasing/falling through.
							return pawn.CurJob;
						}
					}

					// 2) If eligible now, force Tend/Rescue immediately.
					if (canTendNow(pawn, heldPatient))
					{
						JobDef def = heldPatient.Downed ? JobDefOf.Rescue : JobDefOf.TendPatient;
						return JobMaker.MakeJob(def, heldPatient);
					}

					// 3) Not eligible yet: pin to medical pipeline (no release; no fall-through).
					const int holdWaitTicks = 60;
					return JobMaker.MakeJob(JobDefOf.Wait, holdWaitTicks);
				}
			}

			// If we're already tending/rescuing a valid *other* target, keep the commitment.
			// (Prevents combat medics from getting stuck self-tending and then drifting into haul/work.)
			Pawn holdPatientForAntiDrift = null;
			if (combatMedic)
			{
				holdPatientForAntiDrift = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);
			}
			if (combatMedic && pawn.CurJob != null && pawn.CurJob.def != null)
			{
				bool holdActive2 = holdPatientForAntiDrift != null;

				// Only run anti-drift if we are NOT currently holding a pipeline patient.
				if (!holdActive2)
				{
					JobDef curDef = pawn.CurJob.def;

					bool alreadyRescueOrTend =
						curDef == JobDefOf.Rescue ||
						curDef == JobDefOf.TendPatient;

					// Use the held patient (pipeline) if present; otherwise this block should do nothing.
					// This avoids referencing the later `patient` local before its declaration.
					if (holdPatientForAntiDrift != null && alreadyRescueOrTend)
					{
						bool isRelevantJobAlreadyForPatient =
							alreadyRescueOrTend &&
							ArgillianThreatPatientTuning.JobIsMedicalForPatient(pawn.CurJob, holdPatientForAntiDrift);

						if (!isRelevantJobAlreadyForPatient)
						{
							bool medicIsHauling = IsHaulJob(pawn.CurJob);
							bool medicIsMealOrConsume = IsMealOrConsumeLikeJob(pawn.CurJob);

							// Early urgent approximation: only depend on properties available here.
							bool heldPatientUrgentEarly =
								holdPatientForAntiDrift.Downed ||
								!holdPatientForAntiDrift.InBed();

							if ((holdPatientForAntiDrift.Downed && (medicIsHauling || medicIsMealOrConsume) && !IsMedicineFetchJob(pawn.CurJob)) ||
								(!holdPatientForAntiDrift.Downed && (heldPatientUrgentEarly) && (medicIsHauling || medicIsMealOrConsume) && !IsMedicineFetchJob(pawn.CurJob)))
							{
								pawn.pather?.StopDead();

								// Don’t StopAll(true) here; it destabilizes tend/rescue ↔ haul/meal/job arbitration.
								// We only prevent further movement drift so the medical job can take over cleanly.
								pawn.jobs?.ClearQueuedJobs();
								ArgrillianMedicalState.MedicTickCache.MarkNow(pawn);
							}
						}
					}
				}

				// Use the rest of your existing logic below unchanged...
				// (No further changes in this patch; the tend/rescue ↔ haul/move loop fix is entirely in the ReleaseForMedic gate.)
			}

			// IMPORTANT: return whatever your original method returned below.
			// (Keeping your original remainder intact.)
			return null;
		}

		private bool canTendNow(Pawn pawn, Pawn patient)
		{
			bool combatMedic = pawn.GetComp<CompArgrillianMedicSettings>()?.combatMedic == true;

			// NEW: Tend/Rescue stickiness / anti-preemption
			// If this patient is already locked in this medic's medical pipeline (hold),
			// then we must not block Tend/Rescue due to medic-distance gating.
			if (combatMedic)
			{
				Pawn heldPatient = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);
				bool holdActive = heldPatient != null && heldPatient == patient;
				if (heldPatient != null && heldPatient == patient)
				{
					int stableTicks = GetPatientStableTicksForTend(patient);
					int requiredStableTicks = patient.Downed ? 0 : patientStableTicksRequired;

					return IsValidTendTarget(patient, pawn) &&
						stableTicks >= requiredStableTicks &&
						CanReserveTendTarget(pawn, patient);
				}
			}

			int stableTicksOuter = GetPatientStableTicksForTend(patient);
			int requiredStableTicksOuter = patient.Downed ? 0 : patientStableTicksRequired;

			if (!patient.Downed && patient.CurJob != null && patient.CurJob.def != null)
			{
				Job j = patient.CurJob;

				// Don’t start Tend if the patient is actively doing combat/mobility jobs
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

				// Allowed job; continue to tend checks
			}

			// Normal (non-held) case keeps the distance gate.
			// BUT for downed patients we should not require a tight positional distance;
			// Rescue/Tend reachability is already validated by IsValidTendTarget(...).
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