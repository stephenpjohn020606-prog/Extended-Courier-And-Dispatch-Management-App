# Implementation Report: Delivery Attempt Tracking And Return to Sender

## Requirements

This update implements a customer absence and delivery attempt handling mechanism.

**1) Delivery Attempts Tracking:** If a courier is dispatched for delivery and the receiver is absent or unreachable, the application tracks each failed attempt up to a fixed maximum of 3 attempts.

**2) Automatic Return to Sender:** If the delivery cannot be completed after 3 attempts, the courier automatically transitions into the 'Returned to Sender' status ending the dispatch cycle.

**3) Successful Delivery Option:** If the customer is available on any attempt prior to reaching the limit, clicking 'Mark Delivered' marks the courier as Delivered as usual.

## Courier Lifecycle

The Courier Status progression follows these transitions:

- Stored --> Ready for Dispatch --> Dispatched --> Delivered (if successful)
- Stored --> Ready for Dispatch --> Dispatched --> Returned to Sender (after 3 failed attempts)

When a package is in the 'Dispatched' status, the user has access to the 'Delivery Failed' button.

## Code Modifications

### CourierStatus.java

Added the RETURNED_TO_SENDER enum value and updated Cantransitionto() so that DISPATCHED can transition to both DELIVERED and RETURNED_TO_SENDER.

    public enum CourierStatus {
        STORED("Stored"),
        READY_FOR_DISPATCH("Ready for Dispatch"),
        DISPATCHED("Dispatched"),
        DELIVERED("Delivered"),
        RETURNED_TO_SENDER("Returned to Sender");

        public boolean Cantransitionto(CourierStatus Next) {
            if (this == Next) return true;
            if (this == STORED) {
                return Next == READY_FOR_DISPATCH;
            } else if (this == READY_FOR_DISPATCH) {
                return Next == DISPATCHED;
            } else if (this == DISPATCHED) {
              return Next == DELIVERED || Next == RETURNED_TO_SENDER;
            } else {
                return false;
            }
        }
    }

### Courier.java

Added fileds for tracking attempts (MAX_ATTEMPTS=3, Deliveryattempts), along with getter,setter and the Recordfailedattempt() method.

    public static final int MAX_ATTEMPTS = 3;
    private int Deliveryattempts = 0;

    public int Getdeliveryattempts() {
        return Deliveryattempts;
    }

    public void Setdeliveryattempts(int Attempts) {
        this.Deliveryattempts = Attempts;
    }

    public void Recordfailedattempt() throws InvalidStatusTransitionException {
        if (this.Status != CourierStatus.DISPATCHED) {
            throw new InvalidStatusTransitionException(
                "Delivery attempts can only be recorded for dispatched couriers."
            );
        }
        this.Deliveryattempts++;
        if (this.Deliveryattempts >= MAX_ATTEMPTS) {
            Updatestatus(CourierStatus.RETURNED_TO_SENDER);
        }
    }

### ViewCouriersPanel.java

Added the Delivery Failed button to the action bar, added 'Returned to Sender' and implemented Handledeliveryfailed():

    private void Handledeliveryfailed() {
        Courier C = Getselectedcourier();
        if (C == null) return;
        if (C.Getstatus() == CourierStatus.DELIVERED) {
            JOptionPane.showMessageDialog(this, "Courier is already Delivered.", "Cannot Fail",
    JOptionPane.WARNING_MESSAGE);
            return;
        }
        if (C.Getstatus() == CourierStatus.RETURNED_TO_SENDER) {
            JOptionPane.showMessageDialog(this, "Courier has already been Returned to Sender.", "Cannot Fail",
    JOptionPane.WARNING_MESSAGE);
            return;
        }
        if (C.Getstatus() != CourierStatus.DISPATCHED) {
            JOptionPane.showMessageDialog(this, "Courier must be 'Dispatched' to record a delivery attempt.",
    "Cannot Fail", JOptionPane.WARNING_MESSAGE);
            return;
        }
        int Nextattempt = C.Get
