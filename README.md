# Diagrams

![Unable to load diagram](https://github.com/CycleOne-Sliet/diagram/blob/master/flow.svg?raw=true)
![Unable to load diagram](https://github.com/CycleOne-Sliet/diagram/blob/master/technicalTimeline.svg?raw=true)

Basically userflow is
- User Signs In
- Scans QR Code to unlock cycle
- Takes the cycle away
- Goes to another stand
- Scans the qr again
- Puts the cycle back

Requirements that arise from this
- A way to get user authenticated: Satisfied via Firebase Auth
- Scanning QR codes: Satisfied via google `MLKit`
- Communicating with stand and App: Satisfied via starting a `http` server on nodeMCU
- Secure communication between stand and Firebase: Adding Encrypted token sending system with App as a medium.

Business logic for each step as follows:
##### User Signs In

- Just calls `Firebase Auth`'s `signInWithEmailAndPassword` method

##### Scan QR Code to unlock cycle:

- App: Checks if user does not have a cycle to its name already, if yes, go to the procedure for returning the cycle.
- App: Camera is opened
- App: QR Code is parsed
- App: Check QR Code for valid MAC Address.
- App: Tries Connecting to the MAC from the QR Code
- App: Makes a `GET` request to get the current status as an encrypted token.
- App: Makes a `GET` request to get the current status as Non Encrypted JSON Data.
- Stand: Receives the `GET` request and initializes the Rfid reader.
- Stand: Reads the data from rfid reader 5 times to check for redundancy and sends it back after encrypting it.
- App: Verifies that stand does have a cycle associated with it.
- App: Sends the data received in the earlier step and forwards it to Cloud Run function `GetToken`.
- `CloudRun` (`GetToken`): Verifies the request is correct and refers to an actual user and cycle.
- `CloudRun` (`GetToken`): Saves the new info that the user has the cycle, and the stand does not.
- `CloudRun` (`GetToken`): Sends back another encrypted token with additional info (timestamp) intended for the Stand.
- App: Receives the data from `CloudRun` and forwards it to the stand along with the user id, by making a POST request to it.
- Stand: Receives the request data from the App and verifies it by trying to decrypt it and matching it against known values(rfid value, timestamp, user id)
- Stand: If all checks out, Unlock the cycle.

##### Takes the cycle away

- Nothing on the technical side, user burns some calories

##### Scans the qr again

- App: Checks if the user even has a cycle to its name already, if not, go the procedure of getting a cycle unlocked for the user.
- App: Camera is opened
- App: QR Code is parsed
- App: QR Code is checked to be a valid MAC Address
- App: Tries Connecting to the MAC that was in the QR Code
- App: Makes a `GET` request to get the current status as an encrypted token.
- App: Makes a `GET` request to get the current status as Non Encrypted JSON Data.
- Stand: Receives the `GET` request and initializes the Rfid reader.
- Stand: Reads the data from rfid reader 5 times to check for redundancy and sends it back after encrypting it.
- App: Checks that stand does not have a cycle associated with it OR the cycle associated with it is the same one as the one the user is associated with.
- App: If the stand already has the user's cycle, send the current token to the `PutToken` Cloud Function, else continue
- App: Sends the data received in the earlier step and forwards it to Cloud Run function `GetToken`.
- App: Verifies that the request is correct and user does have a cycle associated with him and the request indicates that there is no cycle associated with the stand.
- `CloudRun` Function: Sends back another encrypted token with additional info (timestamp) intended for the Stand.
- App: Receives the data from `CloudRun` and forwards it to the stand along with the user id, by making a POST request to it.
- Stand: Receives the request data from the App and verifies it by trying to decrypt it and matching it against known values(rfid value, timestamp, user id)
- App: Periodically makes get requests at the rate of 1 request per 0.5s to get the new status from stand where the cycle is in the stand for 10 seconds.
- App: When the new status is received, send the new status back to `PutToken` endpoint.
- `CloudRun` (`PutToken`): Verifies the request is correct and refers to an actual user and cycle.
- `CloudRun` (`GetToken`): Saves the new info that the user no longer has the cycle, and the stand does.
- `CloudRun` (`GetToken`): Sends back success code.

#### Data Formats

ServerResponse: Intended to be only readable via the stand, structured as follows

Data Packet obtained by encrypting the following structure:
| `userIdLen` (1 byte) | `cycleIdLen`(1 byte) | `userId` (`userIdLen` bytes) | `cycleId` (`cycleIdLen` bytes) | standMacAddress (6 byte) | time (8 byte) (big endian) |
| -- | -- | -- | -- | -- | -- |

| `IV` (16 bytes) | `Data Packet` ((userIdLen + cycleIdLen) + 32 - (userIdLen + cycleIdLen + 14) % 16)
| -- | -- |


