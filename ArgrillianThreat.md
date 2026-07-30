	private bool canTendNow(Pawn pawn, Pawn patient)
	{
		bool combatMedic = pawn.GetComp<CompArgrillianMedicSettings>()?.combatMedic == true;

		if (pawn == null || patient == null) return false;
		if (patient.Dead) return false;

		// TRACE: entering tend evaluation for this held/assigned target.
		JobGiver_ArgrillianThreatResponse.TraceMedKit(
			"canTendNow_enter",
			pawn,
			patient,
			tendEligibleNow: false,
			retreatingHeldPatient: false
		);

		// KEEP-ACTIVE-JOB GUARD:
		// If the medic is already running the correct Tend/Rescue job for the held/assigned patient,
		// do not let transient movement/stability/distance flips cause the job g iver to abandon it.
		Job cur = pawn.CurJob;
		if (cur != null && cur.def != null)
		{
			if (cur.def == JobDefOf.Rescue)
			{
				// Rescue expects the patient in one of the targets (vanilla/patch dependent).
				if ((cur.targetA.IsValid && cur.targetA.Thing == patient) ||
					(cur.targetB.IsValid && cur.targetB.Thing == patient) ||
					(cur.targetC.IsValid && cur.targetC.Thing == patient))
					return true;
			}

			if (cur.def == JobDefOf.TendPatient)
			{
				if ((cur.targetA.IsValid && cur.targetA.Thing == patient) ||
					(cur.targetB.IsValid && cur.targetB.Thing == patient) ||
					(cur.targetC.IsValid && cur.targetC.Thing == patient))
					return true;
			}
		}

		// Combat medics must start Tend/Rescue immediately for downed patients.
		if (combatMedic && patient.Downed)
		{
			bool ok = IsValidTendTarget(patient, pawn);
			JobGiver_ArgrillianThreatResponse.TraceMedKit(
				"canTendNow_downedFastPath",
				pawn,
				patient,
				tendEligibleNow: ok,
				retreatingHeldPatient: false
			);
			return ok;
		}

		// Held patient invariant: if this patient is locked into this medic's medical pipeline,
		// tend/rescue eligibility should NOT be blocked by the patient's current combat job.
		//
		// FIX for your symptom:
		// When the patient briefly transitions into consume/meal/haul queued while Tend should be pending,
		// stability/eligibility flapping can temporarily drive tendEligibleNow=false and cause job-abandon.
		// While held-for-tend, we treat stability as satisfied (stationary enforcement is owned by PatientMedicHold),
		// and we do not require the tend-stability tick counter.
		if (combatMedic)
		{
			Pawn heldPatient = ArgrillianMedicalState.PatientMedicHold.GetHeldPatient(pawn);

			bool holdActive =
				heldPatient != null &&
				patient != null &&
				heldPatient.thingIDNumber == patient.thingIDNumber;

			if (holdActive)
			{
				// Held-for-tend arbitration guard is the authority; stability ticks must not override it.
				int stableTicksOuter = GetPatientStableTicksForTend(patient);
				int requiredStableTicksOuter = patient.Downed ? 0 : patientStableTicksRequired;

				// Preserve the existing tuning for the "injured" threshold, but do not let stability be the failure gate.
				if (!patient.Downed)
				{
					float hpPct = patient.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
					if (hpPct <= combatMedicInjuredHPPercentThreshold)
						requiredStableTicksOuter = 0;
				}

				bool distanceOk = patient.Downed ? true : pawn.Position.DistanceTo(patient.Position) <= combatTendMaxDistance;

				bool isValid = IsValidTendTarget(patient, pawn);
				bool canReserve = CanReserveTendTarget(pawn, patient);

				// HARD OVERRIDE: if held-for-tend, ignore stability counter for Tend/Rescue eligibility.
				bool stableOk = true;

				if (patient.Downed)
				{
					bool okHeld =
						distanceOk &&
						isValid &&
						stableOk; // stability ignored while held

					JobGiver_ArgrillianThreatResponse.TraceMedKit(
						"canTendNow_heldInvariant_downedFinalGates",
						pawn,
						patient,
						tendEligibleNow: okHeld,
						retreatingHeldPatient: false
					);

					return okHeld;
				}

				bool ok3 = distanceOk && isValid && stableOk && canReserve;

				JobGiver_ArgrillianThreatResponse.TraceMedKit(
					"canTendNow_heldInvariant_injuredFinalGates",
					pawn,
					patient,
					tendEligibleNow: ok3,
					retreatingHeldPatient: false
				);

				// Extra visibility for the exact flapping gate: show stability counter values,
				// even though stability is forced-true when held.
				Verse.Log.Message(
					$"[ArgrillianThreat][TRACE] canTendNow_heldStabilityOverride medic={pawn.thingIDNumber} patient={patient.thingIDNumber} " +
					$"stableTicksOuter={stableTicksOuter} requiredStableTicksOuter={requiredStableTicksOuter} stableOkForced=true");

				return ok3;
			}
		}

		// Normal (not-held) case keeps the stricter checks you already had.
		int stableTicksOuter2 = GetPatientStableTicksForTend(patient);
		int requiredStableTicksOuter2 = patient.Downed ? 0 : patientStableTicksRequired;

		// NEW: combat medics should tend injured targets immediately enough (skip stability wait)
		// to prevent “stand briefly while hostile awareness transitions” and to ensure retreat/tend flow starts.
		if (combatMedic && patient != null && !patient.Downed)
		{
			float hpPct = patient.health?.summaryHealth?.SummaryHealthPercent ?? 1f;
			if (hpPct <= combatMedicInjuredHPPercentThreshold)
				requiredStableTicksOuter2 = 0;
		}

		// If the medic is treating this as an urgent injured target, don't block Tend start just because
		// the patient is still in a combat-like job state.
		bool isUrgentInjuredTarget = combatMedic &&
			patient != null &&
			!patient.Downed &&
			patient.health != null &&
			patient.health.summaryHealth.SummaryHealthPercent <= urgentTendHealthPercent;

		if (isUrgentInjuredTarget)
		{
			requiredStableTicksOuter2 = 0;
		}

		if (patient.Downed)
		{
			bool ok = stableTicksOuter2 >= requiredStableTicksOuter2 &&
				IsValidTendTarget(patient, pawn);

			JobGiver_ArgrillianThreatResponse.TraceMedKit(
				"canTendNow_downed_finalGates",
				pawn,
				patient,
				tendEligibleNow: ok,
				retreatingHeldPatient: false
			);

			return ok;
		}

		bool distanceOk2 = pawn.Position.DistanceTo(patient.Position) <= combatTendMaxDistance;

		bool isValid2 = IsValidTendTarget(patient, pawn);
		bool canReserve2 = CanReserveTendTarget(pawn, patient);
		bool stableOk2 = stableTicksOuter2 >= requiredStableTicksOuter2;

		bool ok2 = distanceOk2 &&
			isValid2 &&
			stableOk2 &&
			canReserve2;

		// NEW: explicit breakdown when Tend is NOT eligible (this is what we need next).
		if (!ok2)
		{
			JobGiver_ArgrillianThreatResponse.TraceMedKit(
				"canTendNow_injured_gateBreakdown",
				pawn,
				patient,
				tendEligibleNow: ok2,
				retreatingHeldPatient: false
			);

			// Use existing TraceMedKit channel only; embed details in the same trace line format.
			Verse.Log.Message(
				$"[ArgrillianThreat][TRACE] canTendNow_gateDetails medic={pawn.thingIDNumber} patient={patient.thingIDNumber} " +
				$"hp={patient.health?.summaryHealth?.SummaryHealthPercent ?? -1f:F2} " +
				$"distanceOk={distanceOk2} dist2={pawn.Position.DistanceTo(patient.Position):F2} maxDist={combatTendMaxDistance:F2} " +
				$"isValidTarget={isValid2} stableTicksOuter={stableTicksOuter2} requiredStableTicksOuter={requiredStableTicksOuter2} stableOk={stableOk2} " +
				$"canReserveTendTarget={canReserve2}"
			);
		}

		JobGiver_ArgrillianThreatResponse.TraceMedKit(
			"canTendNow_injured_finalGates",
			pawn,
			patient,
			tendEligibleNow: ok2,
			retreatingHeldPatient: false
		);
		return ok2;
	}
