This github contains assignemts 1, 2 and 3 of the course Blockchain engineering of the TU Delft, where we have to implement some things in IPV8. 
Documentation on IPV8 can be found for example here: https://py-ipv8.readthedocs.io/en/latest/

#Assignment 1#
The goal is to: 
1. Connects to a running server via the IPv8 peer-to-peer network
2. Computes a Proof of Work (PoW) over your email **and** your GitHub repo URL
3. Submits the solution as an IPv8 message
4. Receives a response from the server (accepted or rejected)

For this, the code 

1. P2P Network Server Discovery
Overlay Joining: The script initializes an IPv8 instance and joins a specific target community identified by the Hex ID 2c1cc6e35ff484f99ebdfb6108477783c0102881.
Peer Discovery: It utilizes standard bootstrap definitions and a RandomWalk strategy to actively discover peers in the network.
Server Target Selection: A scheduled task (check_for_server) runs every 2.0 seconds, looping through discovered peers to match their public key against the hardcoded server public key:
4c69624e61434c504b3a86b23934a28d669c390e2d1fc0b0870706c4591cc0cb178bc5a811da6d87d27ef319b2638ef60cc8d119724f4c53a1ebfad919c3ac4136c501ce5c09364e0ebb

2. The (PoW) Computation
This PoW was requested in the assignment

3. Solution Submission via IPv8 Messaging and Server Response Handling
Once the server peer is successfully discovered, the client cancels its search task and invokes send_greeting.
The data is packed into a custom MyMessagePayload (msg_id = 1) using formats explained in the assignment.
The message is transmitted to the target server peer using self.ez_send().
The community registers an incoming message handler for ServerResponsePayload (msg_id = 2).
When the server transmits the evaluation outcome back to our peer, the on_server_response callback triggers, printing the verification message directly to the terminal output.

Run the code by putting in the terminal
python assignment1.py

#Assignment 2
The assignment is to:
You and two teammates build IPv8 clients that sign challenges from a server within a strict shared budget.
Each round, the server issues a 32-byte nonce. All 3 members sign it; one collects the 3 signatures and submits the bundle. 
Across 3 rounds, each must be submitted by a different member.
**All 3 rounds must finish inside 10 seconds wall-clock**, measured from the moment the server sends the round-1 nonce. 
Faster groups earn bonus credit — the lab is graded on speed, not just correctness.

1. Group Registration
Targeting the Server: The node seeks out the verified server peer matching the unique public key hash LAB2_SERVER_PUBLIC_KEY_HEX.
Registration Submission: Once the connection is stable, one or more group members submit a RegisterPayload containing the strict canonical order of all three team member public keys.
Group Identification: Upon success, the server returns a RegistrationResponsePayload providing a static group_id, authorizing the team to progress to the challenge phase.

 2: Challenge 
The submisser changes across round acording to a fixed SUBMIT_ORDER = [1, 2, 3].
The submitter peer of that round requests a distinct round challenge via ChallengeRequestPayload.
Server Response: The server returns a cryptographic nonce and corresponding metadata through a ChallengeResponsePayload.
Internal Peer Broadcast: The submitter signs the nonce locally, then distributes the raw nonce directly to the other teammates using a NoncePayload message over the P2P overlay.
Signature Generation & Back-Propagation: Non-submitting teammates receive the NoncePayload and verify its round metadata.
Each teammate generates a local elliptic curve cryptographic signature using self.my_peer.key.signature(nonce).
They transmit this back to the round submitter using a SignatureSharePayload.

3: Signature Aggregation & Submission
Validation: The round submitter collects incoming signature shares and evaluates their cryptographic integrity using verify_share().
Bundle Submission: Once all three unique, valid signatures (Member 1, Member 2, and Member 3) are assembled inside the ActiveRound tracking object, the submitter wraps them in a SignatureBundlePayload and sends it to the server.
Round Progression: The server evaluates the bundle, issues a RoundResultPayload, and the group continues to a next round (updates current_round) until all three rounds complete successfully. Teammates stay in sync via RoundDonePayload notifications.

The code can be run by typing in the terminal:
python assignment2.py --key assignmentskey.pem --port 8091
Where assinmentskey.pem is the private key that is registered to the server in assignment 1.

#Assignment 3
The assignment is to:
You and your two teammates from Lab 2 build IPv8 nodes that together run a 3-node Proof-of-Work blockchain. Each member runs one node. Your nodes must mine blocks, propagate them, converge on a single chain, and answer queries from the Lab 3 server.
Once you register, the Lab 3 server joins your blockchain community, submits a test transaction, and walks every chain to check PoW, header linking, body commitment, and 3-way consistency. Your group passes the first time those checks all hold.


