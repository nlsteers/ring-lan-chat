/* JLANCHAT */
package JLANChat;

/* TO DO: ------------------------------------------------------------------- */
// - It's definately easier to keep a separate to-do list
/* -------------------------------------------------------------------------- */
/* Changelist (rough) ------------------------------------------------------- */
// - Many, many things have changed...
/* -------------------------------------------------------------------------- */
/**
 *
 * @authors Christopher Smith, Nathan Steers
 * @createdate Jan 28, 2011
 * @version 0.81
 */
import java.util.*;
import java.awt.*;
import java.awt.event.*;
import gnu.io.*;
import java.io.*;
import javax.swing.JLabel;
import javax.swing.JPanel;

public class JavaLAN {
    /*                                                          10 Byte
    START MARKER >> { | Destination | Source | Type | <----- Payload ------> | Check Sum | } << END MARKER
     *
     * DESTINATION: User ID at destination station (A-Z)
     * SOURCE: User ID at source station (A-Z)
     * PACKET TYPE: L - Login
     *              X - Logout
     *              R - Identification response to login packet
     *              D - Data payload
     *              Y - ACK
     *              N - NAK
     */

    //Rx, Tx and Kbd packets are initialised as global
    private static char[] rxpacket = new char[16];
    private static char[] txpacket = new char[16];
    private static char[] kbdpacket = new char[16];
    //This stations ID
    public static char myaddress = 0;
    private static int pendindex = 0;

    //This method is called to reset a packet so it can be populated with
    //something new. It takes a char array as a parameter.
    private static void clearPacket(char[] packet) {
        int i = 0;
        packet[i++] = '{';
        packet[i++] = '0';
        packet[i++] = myaddress;
        packet[i++] = '0';
        while (i < 15) {
            packet[i++] = ' ';
        }
        packet[i++] = '}';
    }

    //This method is called to calculate and set the checksum of a packet
    //packet[14] before transmission. It takes a char array (packet)
    //as a parameter.
    private static void setCheckSum(char[] packet) {
        int checksum = 0;
        for (int i = 0; i < packet.length; i++) {
            checksum = (checksum + packet[i]) & 0xFF;
        }
        checksum = ((checksum ^ 0xFF) + 1) & 0xFF;
        packet[14] = (char) checksum;
    }

    //This method is called to recalculate the check sum of a packet so it can
    //be compared to the checksum set in packet[14]
    private static char checkCheckSum(char[] packet) {
        int checksum = 0;
        char oldcs = packet[14];
        packet[14] = ' ';
        for (int i = 0; i < packet.length; i++) {
            checksum = (checksum + packet[i]) & 0xFF;
        }
        checksum = ((checksum ^ 0xFF) + 1) & 0xFF;
        packet[14] = oldcs;
        return (char) checksum;
    }

    //Main method for Java LAN
    public static void main(String[] args) {
        //Kbd task switch case values
        final int LOGIN = 0, ADDRPENDING = 1, MENU = 2, GETADDR = 3, INPUTMESS = 4, LOGOUT = 5, PRELOGIN = 6, FLOGIN = 7;
        //Rx task switch case values
        final int WAITING = 0, RECEIVING = 1, ARRIVED = 2, DECODING = 3;
        //Initial Kbd and Rx switch case values and keycount for the
        int kbdcasevalue = LOGIN, rxcasevalue = WAITING, keycount = 4;
        int ltout = 0;
        char key = 0;
        char cRx;
        //senabled = S Case enabled?
        boolean senabled = false, loggingin = false;
        String msg = "";

        //Initialises term, serial port handler and pending table manager
        Term terminal = new Term();
        SerialPortHandler sio = new SerialPortHandler(SerialPortHandler.COM2);
        Administrator admin = new Administrator();

        terminal.println("JLANChat V1.0 RC\nSelect a User ID: ");

        //Start of the super-loop
        while (true) {

            /* (0) KEYBOARD TASK -------------------------------------------- */

            switch (kbdcasevalue) {
                case PRELOGIN:
                    //This case is used to display a message when the user logs
                    //out. It also clears all the addresses in the pend table.
                    terminal.clear();
                    terminal.println("Successfully logged out\nSelect a User ID:");
                    terminal.clearAddress();
                    kbdcasevalue = LOGIN;
                    break;
                case FLOGIN:
                    //This case is used when a user tries logging in with a
                    //duplicate ID. It displays the error message before prompting
                    //for login.
                    terminal.clear();
                    terminal.println("Duplicate ID please choose another ID: ");
                    terminal.clearAddress();
                    kbdcasevalue = LOGIN;
                    break;
                case LOGIN:
                    //If user types a login id send out a login packet
                    if (terminal.getkbhit()) {

                        key = Character.toUpperCase(terminal.getChar());
                        if (key >= 'A' && key <= 'Z') {
                            myaddress = key;
                            terminal.println("Chosen ID (unresolved): " + myaddress + "\n");
                            clearPacket(kbdpacket);
                            kbdpacket[1] = myaddress;
                            kbdpacket[2] = myaddress;
                            kbdpacket[3] = 'L';
                            setCheckSum(kbdpacket);
                            //This if statement can stop the program from logging
                            //in as if the packet space in the pendtable is full
                            //it won't send the login packet.
                            if (admin.getPending(myaddress) == 0) {
                                admin.savePacket(myaddress, kbdpacket);
                                admin.setPending(myaddress, 1);
                                admin.setITXD(myaddress, 0);
                                ltout = 0;
                                kbdcasevalue = ADDRPENDING;
                                loggingin = true;
                            }
                        }
                    }
                    break;
                case ADDRPENDING:
                    //Checks if the user has been logged in by the packet handler
                    //(received a L packet back) every iteration increments the
                    //timeout.
                    if (admin.getLogin(myaddress) == -1) {
                        //If the user has been logged in the menu is displayed and
                        //the user can choose options from the menu.
                        loggingin = false;
                        terminal.clear();
                        String idOkMsg = "Your ID has been set to: " + myaddress;
                        terminal.setAddress(myaddress);
                        terminal.println(idOkMsg);
                        String menuMsg = "Menu:\n"
                                + " (D)est - choose destination ID and type a message\n"
                                + " (S)end - send a message after chosing (D)\n"
                                + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                + " (L)ogout - logout from the ring LAN";
                        terminal.println(menuMsg);
                        kbdcasevalue = MENU;
                    } else {
                        //Timeout incremented
                        ltout++;
                        if (ltout > 100000000) {
                            //Timeout reached program returns back to login stage
                            terminal.println("Login operation timed out...\n"
                                    + "Either ring LAN broken or duplicate login ID");
                            kbdcasevalue = LOGIN;
                            admin.setPending(myaddress, 0);
                            //Resets address back to zero.
                            myaddress = 0;
                        }
                    }
                    break;
                case MENU:
                    //In this case the user can choose options from the menu
                    if (terminal.getkbhit()) {
                        key = Character.toUpperCase(terminal.getChar());
                        switch (key) {
                            case 'D':
                                //Choose a destination to send a message to.
                                //Once chosen the user can enter a message.
                                //Once the message has been typed the program
                                //returns to menu.
                                terminal.clear();
                                terminal.println("Destination ID: ");
                                int[] active = admin.returnLoggedIn();
                                terminal.displayLoggedIn(active);
                                kbdcasevalue = GETADDR;
                                senabled = true;
                                break;
                            case 'S':
                                //This case is only enabled when a message has been
                                //typed using the D case. This case saves the message
                                //to the pending table pending transmission in the Tx
                                //task.
                                if (senabled == true) {
                                    setCheckSum(kbdpacket);
                                    terminal.println("");
                                    if (admin.getPending(kbdpacket[1]) == 0) {
                                        admin.savePacket(kbdpacket[1], kbdpacket);
                                        admin.setPending(kbdpacket[1], 5);
                                        terminal.printMsg(myaddress, kbdpacket[1], msg);
                                        //Program then returns to menu
                                        terminal.clear();
                                        String menuMsg = "Message sent!\nMenu:\n"
                                                + " (D)est - choose a destination ID and type a message\n"
                                                + " (S)end - send a message after chosing (D)\n"
                                                + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                                + " (L)ogout - logout from the ring LAN";
                                        terminal.println(menuMsg);
                                    }
                                }
                                kbdcasevalue = MENU;
                                senabled = false;
                                break;
                            case 'C':
                                //The C case cancels the message that has been
                                //typed.
                                clearPacket(kbdpacket);
                                terminal.clear();
                                //Returns to menu.
                                String menuMsg = "Message cancelled!\nMenu:\n"
                                        + " (D)est - choose a destination ID and type a message\n"
                                        + " (S)end - send a message after chosing (D)\n"
                                        + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                        + " (L)ogout - logout from the ring LAN";
                                terminal.println(menuMsg);
                                kbdcasevalue = MENU;
                                break;
                            case 'L':
                                //Logout menu case - first confirms logout
                                terminal.println("\nAre you sure you want to logout? y/n");
                                kbdcasevalue = LOGOUT;
                                break;
                            default:
                                break;
                        }
                    }
                    break;
                case LOGOUT:
                    //Logout case waits for a Y/N char - starts logout process if
                    // Y is typed and aborts, returns to menu is N is typed.
                    if (terminal.getkbhit()) {
                        key = Character.toUpperCase(terminal.getChar());
                        switch (key) {
                            case 'Y':
                                terminal.println("Logging out...");
                                clearPacket(kbdpacket);
                                kbdpacket[1] = myaddress;
                                kbdpacket[2] = myaddress;
                                kbdpacket[3] = 'X';
                                setCheckSum(kbdpacket);
                                //sends out a logout packet
                                if (admin.getPending(myaddress) == 0) {
                                    admin.savePacket(myaddress, kbdpacket);
                                    admin.setPending(myaddress, 1);
                                }
                                break;
                            case 'N':
                                terminal.println("");
                                terminal.clear();
                                String menuMsg = "Logout aborted.\nMenu:\n"
                                        + " (D)estination - choose destination ID and type a message\n"
                                        + " (S)end - send a message after chosing (D)\n"
                                        + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                        + " (L)ogout - logout from the ring LAN";
                                terminal.println(menuMsg);
                                kbdcasevalue = MENU;
                                break;
                            default:
                                break;
                        }
                    }
                    break;
                case GETADDR:
                    //This case waits for (or handles) the address of the destination
                    //staion to be typed for the user
                    if (terminal.getkbhit()) {
                        key = Character.toUpperCase(terminal.getChar());
                        //Checks if the character chosen is within the A-Z range.
                        if (key >= 'A' && key <= 'Z') {
                            //Checks if the chosen destination is currently logged in.
                            //using the pendtable. If destination not logged in an
                            //error message is displayed and the program returns to the menu.
                            if (admin.getLogin(key) == 0) {
                                terminal.clear();
                                terminal.println("Chosen destination not logged in at present\n");
                                String menuMsg = "Menu:\n"
                                        + " (D)estination - choose destination ID and type a message\n"
                                        + " (S)end - send a message after chosing (D)\n"
                                        + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                        + " (L)ogout - logout from the ring LAN";
                                terminal.println(menuMsg);
                                senabled = false;
                                kbdcasevalue = MENU;
                            } else {
                                //Initialisation of a D (message) packet
                                clearPacket(kbdpacket);
                                kbdpacket[1] = key;
                                kbdpacket[3] = 'D';
                                keycount = 4;
                                kbdcasevalue = INPUTMESS;
                                terminal.println("Send to: " + kbdpacket[1]);
                                terminal.println("Please input a message: \n");
                            }
                        } else {
                            //Returns an error message if an invalid key is pressed
                            //and goes back to the menu.
                            terminal.clear();
                            terminal.println("Invalid key pressed\n");
                            String menuMsg = "Menu:\n"
                                    + " (D)estination - choose destination ID and type a message\n"
                                    + " (S)end - send a message after chosing (D)\n"
                                    + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                    + " (L)ogout - logout from the ring LAN";
                            terminal.println(menuMsg);
                            senabled = false;
                            kbdcasevalue = MENU;
                        }
                    }
                    break;


                case INPUTMESS:
                    //This case handles user input which forms the payload of
                    //D packet. When enter is pressed or the character limited
                    //reached the progam returns to the menu and displays the
                    //message ready to be sent out.
                    if (terminal.getkbhit()) {
                        key = terminal.getChar();
                        if (keycount < 14 && key != '\n') {
                            if (key == 8 && keycount > 4) {
                                kbdpacket[keycount - 1] = 0;
                                keycount--;
                                terminal.del();
                            } else {
                                if(key != 8 && key != 9) {
                                    kbdpacket[keycount++] = key;
                                    terminal.putChar(kbdpacket[keycount - 1]);
                                }
                            }
                        } else {
                            //Extracts the msg payload from the packet to display
                            //on the terminal.
                            char[] cmsg = new char[10];
                            for (int i = 4; i < kbdpacket.length - 2; i++) {
                                cmsg[i - 4] = kbdpacket[i];
                            }
                            msg = new String(cmsg);
                            //Converts char to string.
                            //Returns to menu.
                            terminal.clear();
                            terminal.println("Your message: " + msg);
                            String menuMsg = "Menu:\n"
                                    + " (D)est - choose destination ID and type a message\n"
                                    + " (S)end - send a message after chosing (D)\n"
                                    + " (C)ancel - cancel and return to menu after chosing (D)\n"
                                    + " (L)ogout - logout from the ring LAN";
                            terminal.println(menuMsg);
                            kbdcasevalue = MENU;
                        }
                    }
                    break;

            }
            /* (0) END KEYBOARD TASK ---------------------------------------- */

            /* (1) Tx TASK -------------------------------------------------- */
            //This keeps the pendindex (used to reference the pend arrays) within
            //the A - Z range. This keeps the index cycling around the alphabet.
            //PENDINDEX increments at start
            if (++pendindex > ('Z' - 'A')) {
                pendindex = 0;
            }
            //The Tx task first checks to see if the packet for the current address
            //is pending - if not it moves on. It then checks the delay to see if
            //it equals zero and if it does it sends out the packet.
            if (admin.getPending(pendindex) > 0) {
                if (admin.getITXD(pendindex) > 0) {
                    admin.decITXD(pendindex);
                } else {
                    admin.setITXD(pendindex, 100);
                    sio.sendpacket(admin.getPacket(pendindex));
                    //Prints to debug panel
                    terminal.printDebug("Packet sent: ", admin.getPacket(pendindex));
                    //decrments packet's pending value
                    admin.decPending(pendindex);
                }

            }
            /* (1) END Tx TASK ---------------------------------------------- */

            /* (2) Rx TASK -------------------------------------------------- */
            switch (rxcasevalue) {
                case WAITING:
                    //This is the start of the receive process.
                    //On every loop it gets a char off the Rx buffer and only
                    //progresses if it recieves a '{' opening curly bracket.
                    if (sio.getChar() == '{') {
                        rxcasevalue = RECEIVING;
                    }
                    break;
                case RECEIVING:
                    //This case populates the rxpacket into a full packet by
                    //getting the next 15 characters off the Rx buffer
                    rxpacket[0] = '{';
                    for (int i = 1; i < 16; i++) {
                        rxpacket[i] = sio.getChar();
                    }

                    rxcasevalue = ARRIVED;
                    break;
                case ARRIVED:
                    System.out.print("Packet received: ");
                    System.out.println(rxpacket);
                    //Prints the packet that has arrived to the debug panel
                    terminal.printDebug("Packet received: ", rxpacket);
                    if (rxpacket[3] != 'D') {
                        rxcasevalue = DECODING;
                        break;
                    } else {
                        //Calculates and compares the checksum for any D packets
                        //Gets the checksum from the recieved packet
                        char pcs = rxpacket[14];
                        if (rxpacket[1] == myaddress) {
                            //Calculates the checksum for the received packet
                            char rcs = checkCheckSum(rxpacket);
                            //And compares..
                            if (pcs == rcs) {
                                //IF OK SEND ACK
                                clearPacket(txpacket);
                                txpacket[1] = rxpacket[2];
                                txpacket[3] = 'A';
                                admin.savePacket(rxpacket[2], txpacket);
                                admin.setPending(rxpacket[2], 1);
                                rxcasevalue = DECODING;
                                break;
                            } else {
                                //IF NOT OK SEND NAK
                                clearPacket(txpacket);
                                txpacket[1] = rxpacket[2];
                                txpacket[3] = 'N';
                                terminal.printMsg(rxpacket[2], rxpacket[1], "[Rx ERROR]");
                                admin.savePacket(rxpacket[2], txpacket);
                                admin.setPending(rxpacket[2], 1);
                                clearPacket(rxpacket);
                                rxcasevalue = WAITING;
                                break;
                            }
                        }
                    }
                case DECODING:
                    //TO ME
                    if (rxpacket[1] == myaddress) {
                        // TO ME FROM ME
                        if (rxpacket[2] == myaddress) {
                            switch (rxpacket[3]) {

                                // CASE L (login) - Sets current login to -1 (me)
                                /* If packet is (really) from another station accidently choosen
                                a duplicate ID then this station (if already logged in with that
                                id will think this packet is from itself and harmlessly change
                                its own ID to -1 (me) and destroy the packet, stopping it from
                                getting back originating station and preventing ID duplication */

                                case 'L': // login packet from me
                                    //This either logs the logging in user in or
                                    //If duplicate ID sends out an ACK packet
                                    //Which in turn will log the duplicate station
                                    //out.
                                    if (admin.getLogin(myaddress) == -1) {
                                        clearPacket(txpacket);
                                        txpacket[1] = myaddress;
                                        txpacket[3] = 'A';
                                        admin.savePacket(myaddress, txpacket);
                                        admin.setPending(myaddress, 1);
                                        clearPacket(rxpacket);
                                        rxcasevalue = WAITING;
                                        break;
                                    } else {
                                        admin.setLogin(myaddress, -1);
                                        clearPacket(rxpacket);
                                        rxcasevalue = WAITING;
                                        break;
                                    }
                                case 'X':
                                    //Logout packet completed loop - logs station
                                    //out and returns the kbd task to the login state.
                                    admin.setLogin(rxpacket[2], 0);
                                    myaddress = 0;
                                    clearPacket(rxpacket);
                                    kbdcasevalue = PRELOGIN;
                                    rxcasevalue = WAITING;
                                    break;
                                case 'D':
                                    //Message packet to myself - dont display
                                    //packet (already displayed when sent originally.
                                    char[] cmsg = new char[10];
                                    for (int i = 4; i < rxpacket.length - 2; i++) {
                                        cmsg[i - 4] = rxpacket[i];
                                    }
                                    msg = new String(cmsg);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'A':
                                    //Logs user out if trying to log in - else
                                    //Stops pending Re-Tx packets.
                                    if(loggingin == true) {
                                        kbdcasevalue = FLOGIN;
                                    }
                                    admin.setPending(rxpacket[2], 0);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'N':
                                    //NAK for myself - resend the packet
                                    admin.setPending(rxpacket[1], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                default:
                                    //Default should handle any illegal values
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                            }
                        }
                        // FROM SOMEONE ELSE TO ME
                        if (rxpacket[2] != myaddress) {
                            switch (rxpacket[3]) {

                                // CASE R (login response) - Response to login packet, read packet update pendtable
                                case 'R':
                                    admin.setLogin(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'D':
                                    //Message receieved, print message.
                                    char[] cmsg = new char[10];
                                    for (int i = 4; i < rxpacket.length - 2; i++) {
                                        cmsg[i - 4] = rxpacket[i];
                                    }
                                    msg = new String(cmsg);
                                    terminal.printMsg(rxpacket[2], rxpacket[1], msg);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'A':
                                    //ACK to me from someone else - successful
                                    //msg, set pending to zero.
                                    admin.setPending(rxpacket[2], 0);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'N':
                                    //NAK to me from someone else - unsucessful
                                    //message, Re-Tx message
                                    admin.setPending(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                default:
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                            }
                        }
                    }
                    //TO SOMEONE ELSE
                    if (rxpacket[1] != myaddress) {
                        // TO SOMEONE ELSE FROM ME
                        if (rxpacket[2] == myaddress) {
                            switch (rxpacket[3]) {

                                // CASE R (login response) - Response to login packet, read packet update pendtable
                                case 'R':
                                    //Stop response packet, set dest to logged
                                    //in in pending table.
                                    admin.setLogin(rxpacket[1], 0);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                case 'D':
                                    //Message packet failed to go to destination
                                    //stop packet from going round the loop.
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'A':
                                    //Receive failed, stop packet.
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'N':
                                    //Receive failed, stop packet.
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                default:
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;

                            }
                        }
                        // TO SOMEONE ELSE FROM SOMEONE ELSE
                        if (rxpacket[2] != myaddress) {
                            switch (rxpacket[3]) {

                                // CASE L (login) - Send back out, update pendtable Tx login response
                                /* NOTE! packets should be saved the pending table rather than directly
                                sent out -- It doesnt block but something to think about... */

                                case 'L': // login packet from someone else
                                    //Relay login packet back to source, add
                                    //address to active user list (if station is
                                    //logged in)
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    admin.setLogin(rxpacket[2], 1);
                                    clearPacket(txpacket);
                                    if (myaddress != 0) {
                                        txpacket[1] = rxpacket[2];
                                        txpacket[2] = myaddress;
                                        txpacket[3] = 'R';
                                        admin.savePacket(myaddress, txpacket);
                                        admin.setPending(myaddress, 1);
                                    }
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'D':
                                    //relay message packet
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'X':
                                    //relay logout packet, remove from active
                                    //user list.
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    admin.setLogin(rxpacket[2], 0);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'R':
                                    //relay login response packet, update user
                                    //list with the packet destination
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    admin.setLogin(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'A':
                                    //relay ACK
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                case 'N':
                                    //relay NAK
                                    admin.savePacket(rxpacket[2], rxpacket);
                                    admin.setPending(rxpacket[2], 1);
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                                default:
                                    clearPacket(rxpacket);
                                    rxcasevalue = WAITING;
                                    break;
                            }
                        }
                    }

                    break;
            }
            /* (2) END Rx TASK ---------------------------------------------- */
        }
    }
}

class SerialPortHandler {

    public static final String COM1 = "COM1", COM2 = "COM2", COM3 = "COM3", COM4 = "COM4",
            COM7 = "COM7", COM8 = "COM8", // Common windows COM ports
            TTYS0 = "/dev/ttyS0", TTYS1 = "/dev/ttyS1", // Linux standard COM1 and COM2
            TTYUSB0 = "/dev/ttyUSB0", TTYUSB1 = "/dev/ttyUSB1", // Linux standard USB-Serial
            OSX0 = "/dev/ttys000", OSX1 = "/dev/ttys001"; // Emulated OSX Ports
    private Enumeration portList;
    private CommPortIdentifier portId;
    private SerialPort serialPort = null;
    private OutputStream outputStream;
    private InputStream inputStream;

    public SerialPortHandler(String comStr) {

        System.out.println();
        System.out.println("Checking what ports are available:-");
        portList = CommPortIdentifier.getPortIdentifiers();

        while (portList.hasMoreElements()) {
            portId = (CommPortIdentifier) portList.nextElement();
            System.out.print("" + portId.getName());
            if (portId.getPortType() == CommPortIdentifier.PORT_SERIAL) {
                if (portId.getName().equals(comStr)) {
                    try {
                        serialPort = (SerialPort) portId.open("java lan", 1000);
                        serialPort.setSerialPortParams(9600, serialPort.DATABITS_8, serialPort.STOPBITS_1, serialPort.PARITY_NONE);
                        serialPort.enableReceiveThreshold(0);
                        serialPort.setRTS(true);
                        serialPort.setDTR(true);

                        outputStream = serialPort.getOutputStream();
                        inputStream = serialPort.getInputStream();

                        System.out.println(" port set-up...");
                    } catch (IOException e) {
                    } catch (UnsupportedCommOperationException e) {
                        System.out.println(" Unsupported use...");
                        System.exit(0);
                    } catch (PortInUseException e) {
                        System.out.println(" Port in use...");
                        System.exit(0);
                    }
                }
            }
            System.out.println();
        }
        System.out.println("Finished Checking!");
        if (serialPort == null) {
            System.out.println("No " + comStr + " port found...");
            System.exit(0);
        }
    }

    //GETCHAR gets the next character from the Rx buffer/stream - used to get
    //the key pressed
    public char getChar() {
        char b = (char) -1;
        try {
            b = (char) inputStream.read();
        } catch (IOException e) {
        }
        return b;
    }

    //PUTCHAR writes a character to the Tx buffer/stream [THIS ISN'T USED]
    public void putChar(char b) {
        try {
            outputStream.write(b);
        } catch (IOException e) {
        }
    }

    //SENDPACKET writes each char from the packet one-by-one to the Tx buffer/
    //stream. This is used only in the Tx task in the main program
    public int sendpacket(char[] ppacket) {
        try {
            for (int i = 0; i < ppacket.length; i++) {
                outputStream.write(ppacket[i]);
            }
            System.out.print("Packet sent: ");
            System.out.println(ppacket);
        } catch (IOException e) {
            System.out.println("PACKET NOT SENT!!! " + ppacket);
        }
        return 0;
    }
}

class Term extends Frame implements KeyListener, WindowListener, ActionListener {

    private Frame f;
    private TextArea menu, messages, debugPane;
    private JPanel top, bottom;
    private char[] lastChar = new char[3000];
    private boolean kbhit, debug = false;

    //Dimensions for the various Swing components
    //Some of these dimensions are used when the window is resized when the
    //debug mode is switched in.
    Dimension d1 = new Dimension(400, 600);
    Dimension d1a = new Dimension(400, 450);
    Dimension d2 = new Dimension(670, 600);
    Dimension d3 = new Dimension(250, 600);
    Dimension d3a = new Dimension(250, 450);
    Dimension d4 = new Dimension(670, 25);
    Dimension d5 = new Dimension(652, 150);
    Dimension d6 = new Dimension(670, 175);
    private int kbdbufcount = 0, kbdbufpointer = 0;
    private final char AN = 0;
    private JLabel yourAddr;

    public Term() {
        setLayout(new BorderLayout());
        kbhit = false;
        f = new Frame("JLANCHAT");

        //Top panel
        top = new JPanel();
        top.setPreferredSize(d4);
        JLabel appName = new JLabel("JLANChat VERSION 1.0 RC");
        top.add(appName);
        Font tFont = new Font("Monospaced", Font.PLAIN, 12);
        yourAddr = new JLabel("You are not currently logged in.");
        top.add(yourAddr);

        //Menu textbox
        menu = new TextArea("", 100, 55, TextArea.SCROLLBARS_NONE);
        menu.setFont(tFont);
        menu.addKeyListener(this);
        menu.setEditable(false);
        f.addWindowListener(this);
        menu.setBackground(Color.WHITE);
        menu.setForeground(Color.DARK_GRAY);
        menu.setPreferredSize(d1);

        //Messages textbox
        messages = new TextArea();
        messages.setFont(tFont);
        messages.setEditable(false);
        f.addWindowListener(this);
        messages.setBackground(Color.WHITE);
        messages.setForeground(Color.DARK_GRAY);
        messages.setPreferredSize(d3);

        //Debug panel
        bottom = new JPanel();
        bottom.setPreferredSize(d6);
        JLabel debugTitle = new JLabel("Debug Pane");
        debugPane = new TextArea();
        debugPane.setFont(tFont);
        debugPane.setEditable(false);
        debugPane.setBackground(Color.WHITE);
        debugPane.setForeground(Color.DARK_GRAY);
        debugPane.setPreferredSize(d5);
        bottom.add(debugTitle, BorderLayout.NORTH);
        bottom.add(debugPane, BorderLayout.SOUTH);

        //Frame settings and layouting
        f.add(menu, BorderLayout.WEST);
        f.add(messages, BorderLayout.EAST);
        f.add(top, BorderLayout.NORTH);
        f.setSize(600, 600);
        f.setMaximumSize(d2);
        f.setMinimumSize(d2);
        f.setVisible(true);
    } //end term contructor

    //Used to change the address label when the user logs in
    //Parameters: Address of station
    public void setAddress(char addr) {
        yourAddr.setText("Your current address is: " + addr);
    }

    //Used to reset the address label text when the user logs out
    public void clearAddress() {
        yourAddr.setText("You are not currently logged in.");
    }

    //Used to rearrange the SWING components when the user enters debug mode
    //validate is used to refresh the window and load the new components.
    public void enterDebugMode() {
        menu.setPreferredSize(d1a);
        messages.setPreferredSize(d3a);
        f.add(bottom, BorderLayout.SOUTH);
        f.validate();
    }

    //Rearranges the components when leaving debug mode.
    public void leaveDebugMode() {
        f.remove(bottom);
        menu.setPreferredSize(d1);
        messages.setPreferredSize(d3);
        f.validate();
    }

    public void keyPressed(KeyEvent event) {
    }

    public void keyReleased(KeyEvent event) {
    }

    //KeyTyped is used to get the last key typed (stores them in a buffer which
    //is accessed by the main program in a queue so that keys aren't missed).
    //If tab is pressed - toggles debug mode by calling routines.
    public void keyTyped(KeyEvent event) {
        lastChar[kbdbufcount] = event.getKeyChar();
        if (lastChar[kbdbufcount] == 0x9) {
            if (debug == false) {
                debug = true;
                this.enterDebugMode();
            } else {
                debug = false;
                this.leaveDebugMode();
            }
        }
        kbdbufcount++;
        lastChar[kbdbufcount] = AN;
        kbhit = true;
    }

    public void windowActivated(WindowEvent event) {
    }

    public void windowDeactivated(WindowEvent event) {
    }

    public void windowDeiconified(WindowEvent event) {
    }

    public void windowIconified(WindowEvent event) {
    }

    public void windowClosed(WindowEvent event) {
    }

    //Disposes resources when program closes
    public void windowClosing(WindowEvent event) {
        dispose();
        System.exit(0);
    }

    public void windowOpened(WindowEvent event) {
    }

    public void windowOpening(WindowEvent event) {
    }

    public void actionPerformed(ActionEvent event) {
        System.out.println("I got a call back from " + event.getActionCommand());
    }

    public void print(String s) {
        menu.append(s);
    }

    //Prints to menu textbox
    public void println(String s) {
        menu.append(s + "\n");
    }

    public void putChar(char ch) {
        StringBuffer str = new StringBuffer(" ");
        str.setCharAt(0, ch);
        menu.append(str.toString());
    }

    public boolean getkbhit() {
        return kbhit;
    }

    public char getChar() {
        if (kbhit == true) {
            char c = lastChar[kbdbufpointer];
            kbdbufpointer++;
            if (lastChar[kbdbufpointer] == AN) {
                kbhit = false;
            }
            return c;
        } else {
            return 0;
        }

//        if (kbhit == true) {
//            kbhit = false;
//            return lastChar;
//        } else {
//            return 0;
//        }
    }

    // Clears the terminal screen - I put this in to make the cli
    // look more interactive...
    public void clear() {
        menu.setText("");
    }

    //Routine to handle deleting chars when entering text.
    public void del() {
        String text = menu.getText();
        String newtext = text.substring(0, text.length() - 1);
        menu.setText(newtext);
        menu.setCaretPosition(newtext.length());
    }

    //Prints to the messages pane
    //Paramters: Source address, Destination address, msg
    public void printMsg(char u, char d, String s) {
        char user = u;
        char destination = d;
        String message = s;
        if (user != JavaLAN.myaddress) {
            messages.append(user + " to " + "(me) " + destination + ": " + message + '\n');
        } else {
            messages.append("(me) " + user + " to " + destination + ": " + message + '\n');
        }
    }

    //Prints a horizontal list of all logged in users
    public void displayLoggedIn(int[] active) {
        this.print("Users logged in:");
        for (int i = 0; i < active.length; i++) {
            if (active[i] == 1 || active[i] == -1) {
                int u = i + 'A';
                char au = (char) u;
                au = Character.toUpperCase(au);
                this.print(" " + au);

            }

        }
        this.println("");
    }

    //Prints messages and packets to the debug pane
    //Parameters: Debug message (usually packet sent/received), packet
    public void printDebug(String msg, char[] packet) {
        String s = new String(packet);
        debugPane.append(msg + s + "\n");
    }
}//end term class

class Administrator {

    //Logged in status array
    int[] login = new int[26];
    //Message pending array
    int[] pending = new int[26];
    //Delay array
    int[] itxd = new int[26];
    //Packet store array
    char[][] ppacket = new char[26][16];

    //Pending table/admin constructor initialises everything to 0.
    public Administrator() {
        for (int i = 0; i < login.length; i++) {
            login[i] = 0;
            pending[i] = 0;
            itxd[i] = 0;
        }
    }

    //Used to copy packets to the packet store
    //Parameters: User address (defines index), packet to store
    public void savePacket(char u, char[] packet) {
        int ui = u - 'A'; // ui - User Index... A to 0, B to 1, F to 5 etc...
        for (int i = 0; i < packet.length; i++) {
            ppacket[ui][i] = packet[i];
        }
    }

    //NOT USED... clears the packet from the packet store
    //Parameters: User address/index
    public void wipePacket(char u) {
        int ui = u - 'A';
        for (int i = 0; i < 16; i++) {
            ppacket[ui][i] = 0;
        }
    }

    public int[] returnLoggedIn() {
        return login;
    }

    public int getLogin(char u) {
        return login[u - 'A'];
    }

    public int getPending(char u) {
        return pending[u - 'A'];
    }

    public int getPending(int u) {
        return pending[u];
    }

    public int getITXD(int u) {
        return itxd[u];
    }

    public char[] getPacket(int u) {
        return ppacket[u];
    }

    //Set pending resets ITXD so that the initial packets are not delayed but
    //subsequent Re-Tx packets are
    public void setPending(char u, int i) {
        pending[u - 'A'] = i;
        this.setITXD(u, 0);
    }

    //Overloaded: Set itxd using char addres
    public void setITXD(char u, int i) {
        itxd[u - 'A'] = i;
    }
    //Set itxd using index
    public void setITXD(int u, int i) {
        itxd[u] = i;
    }

    public void setLogin(char u, int i) {
        login[u - 'A'] = i;
    }

    public void decPending(int u) {
        pending[u]--;
    }

    public void decITXD(int u) {
        itxd[u]--;
    }

    //Resets all the logins - used when the user logs out
    public void resetLogin() {
        for (int i = 0; i < login.length; i++) {
            login[i] = 0;
        }
    }
}
