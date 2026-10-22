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
        int Nextattempt = C.Getdeliveryattempts() + 1;
        int Confirm = JOptionPane.showConfirmDialog(
            this,
            "Record failed delivery attempt (" + Nextattempt + "/" + Courier.MAX_ATTEMPTS + ") for " +
    C.Getcourierid() + "?\n(Receiver not present / unreachable)",
            "Confirm Failed Attempt",
            JOptionPane.YES_NO_OPTION
        );
        if (Confirm == JOptionPane.YES_OPTION) {
            try {
                Courier Updated = Db.Recordfailedattempt(C.Getcourierid());
                if (Updated.Getstatus() == CourierStatus.RETURNED_TO_SENDER) {
                    JOptionPane.showMessageDialog(this, "Max attempts reached (" + Courier.MAX_ATTEMPTS +
    "/" + Courier.MAX_ATTEMPTS + ").\nCourier " + Updated.Getcourierid() + " returned back to sender!",
    "Returned to Sender", JOptionPane.WARNING_MESSAGE);
                } else {
                    JOptionPane.showMessageDialog(this, "Delivery attempt " + Updated.Getdeliveryattempts() + "
    of " + Courier.MAX_ATTEMPTS + " recorded.\nCourier remains Dispatched.");
                }
                Dashboard.Refreshalldata();
            } catch (Exception Ex) {
                JOptionPane.showMessageDialog(this, "Error: " + Ex.getMessage(), "Error",
    JOptionPane.ERROR_MESSAGE);
            }
        }
    }

### CourierDetailsDialog.java

Updated the Courier Information grid in the detail dialog to display the delivery attempt count when user inspect a courier.

    JPanel P1 = new JPanel(new GridLayout(5, 2, 8, 4));
    P1.setBorder(BorderFactory.createTitledBorder("Courier Information"));
    P1.add(new JLabel("Courier ID:")); P1.add(new JLabel(C.Getcourierid()));
    P1.add(new JLabel("Courier Type:")); P1.add(new JLabel(C.Getcouriertype() + " (" +
    C.Getprioritylevel() + ")"));
    P1.add(new JLabel("Current Status:")); P1.add(new
    JLabel(C.Getstatus().Getdisplayname()));
    P1.add(new JLabel("Delivery Attempts:")); P1.add(new JLabel(C.Getdeliveryattempts() + " / 
    " + Courier.MAX_ATTEMPTS));
    P1.add(new JLabel("Booking Date:")); P1.add(new JLabel(C.Getbookingdate()));

## Database Changes

The database layer in DBManager.java was updated to support storage and retrieval of delivery attempts

A new integer column named delivery_attempts has been added to the couriers table with a default value of 0.

**Table Schema Definition:**

    CREATE TABLE IF NOT EXISTS couriers (
        courier_id VARCHAR(50) PRIMARY KEY,
        courier_type VARCHAR(50) NOT NULL,
        sender_id INT NOT NULL,
        receiver_id INT NOT NULL,
        destination_address TEXT NOT NULL,
        destination_city VARCHAR(100) NOT NULL,
        weight DOUBLE NOT NULL,
        parcel_type VARCHAR(100) NOT NULL,
        booking_date VARCHAR(50) NOT NULL,
        status VARCHAR(50) NOT NULL,
        storage_location_id INT,
        delivery_charge DOUBLE NOT NULL,
        delivery_attempts INT NOT NULL DEFAULT 0,
        extra_info TEXT,
        FOREIGN KEY (sender_id) REFERENCES customers(customer_id),
        FOREIGN KEY (receiver_id) REFERENCES customers(customer_id),
        FOREIGN KEY (storage_location_id) REFERENCES storage_locations(location_id) ON DELETE
    SET NULL
    );

## Update Logic in DBManager.java

**Dedicated Update Method:**

Recordfailedattempt(String Courierid) handles incrementing and status updating in the database

    public Courier Recordfailedattempt(String Courierid)
            throws CourierNotFoundException, InvalidStatusTransitionException,
    DatabaseOperationException {
        Courier C = Getcourierbyid(Courierid);
        C.Recordfailedattempt();
        try {
            Connection Conn = Getconnection();
            try (PreparedStatement Pstmt = Conn.prepareStatement(
                    "UPDATE couriers SET delivery_attempts = ?, status = ? WHERE courier_id = ?")) {
                Pstmt.setInt(1, C.Getdeliveryattempts());
                Pstmt.setString(2, C.Getstatus().Getdisplayname());
                Pstmt.setString(3, Courierid);
                Pstmt.executeUpdate();
            }
        } catch (Exception E) {
            throw new DatabaseOperationException("Failed to record delivery attempt: " +
    E.getMessage(), E);
        }
        return C;
    }
