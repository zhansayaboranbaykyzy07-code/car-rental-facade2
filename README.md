# Assignment 4: Car Rental Facade Chain System

## Domain & Overview
**Domain:** Variant 13 — Car Rental  
This application implements a **Facade pattern** over three completely independent subsystems:

1. **Subsystem A (Intake / Reservation):**
   - Classes: `FleetTracker`, `PricingEngine`, `ReservationService`, `ReservationRepository`.
   - Entity: `CarReservation` (Statuses: `NEW` -> `RESERVED` -> `CANCELLED`).
2. **Subsystem B (Processing / Insurance & Deposit):**
   - Classes: `InsuranceCalculator`, `DepositHandler`, `InsuranceService`, `InsuranceRepository`.
   - Entity: `InsuranceDeposit` (Statuses: `NEW` -> `HELD` -> `RELEASED`).
3. **Subsystem C (Finalization / Inspection & GPS):**
   - Classes: `DamageInspector`, `GPSTrackerService`, `RentalService`, `RentalRepository`.
   - Entity: `RentalAgreement` (Statuses: `NEW` -> `CONFIRMED` -> `REVOKED`).

Durable execution progress is safely stored in an embedded **SQLite database** via `ChainProgressRepository`.

---

## Cross-Subsystem Consistency & Compensation Depths (Section 3.4)

The `CarRentalFacade` coordinates the sequential chain `A -> B -> C` and guarantees cross-subsystem consistency across failures using persistent state:

1. **Compensation Depth 1 (Failure at Step B):**
   - Step A commits (`RESERVED`), progress saved as `PENDING_B`.
   - If Step B fails (e.g. deposit process rejected), `CarRentalFacade` catches the error and compensates Subsystem A alone (`subsystemA.cancelReservation()`).
   - Persisted status: `COMPENSATED_A`.

2. **Compensation Depth 2 (Failure at Step C):**
   - Steps A and B commit (`RESERVED`, `HELD`), progress saved as `PENDING_C`.
   - If Step C fails (e.g. GPS tracker activation failure), `CarRentalFacade` triggers a **two-step sequential rollback**:
     1. Unwinds Subsystem B: `subsystemB.releaseDeposit()` (`RELEASED`).
     2. Unwinds Subsystem A: `subsystemA.cancelReservation()` (`CANCELLED`).
   - Persisted status: `COMPENSATED_ALL`.

---

## Principle of Least Knowledge (Law of Demeter) (Section 3.5)

### Before Facade (Without Facade)
Without the facade, the client is forced to know about 6+ classes across 3 distinct subsystems, manage execution order, handle specific exceptions, and write manual compensation logic:

```java
// Direct coupling to multiple friends and subsystems:
CarReservation res = fleetTracker.checkAndReserve(carId, days);
double price = pricingEngine.calculate(carId, days);
reservationRepo.save(res);

try {
    InsurancePolicy policy = insuranceCalc.calculatePolicy(driverId);
    boolean depositOk = depositHandler.holdDeposit(driverId, policy.getAmount());
    if (!depositOk) {
        reservationRepo.updateStatus(res.getId(), Status.CANCELLED); // Manual Rollback A
        return;
    }
    insuranceRepo.save(policy);

    try {
        gpsTracker.activate(carId);
        damageInspector.verify(carId);
        rentalRepo.confirmAgreement(res.getId());
    } catch (Exception e) {
        // Manual Rollback B then A
        depositHandler.releaseDeposit(driverId);
        insuranceRepo.updateStatus(policy.getId(), Status.RELEASED);
        reservationRepo.updateStatus(res.getId(), Status.CANCELLED);
    }
} catch (Exception e) { ... }
