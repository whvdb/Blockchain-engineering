This github contains assignemts 1, 2 and 3 of the course Blockchain engineering of the TU Delft, where we have to implement some things in IPV8. 
Documentation on IPV8 can be found for example here: https://py-ipv8.readthedocs.io/en/latest/

# Assignment 1
The goal is to: 
1. Connects to a running server via the IPv8 peer-to-peer network
2. Computes a Proof of Work (PoW) over your email **and** your GitHub repo URL
3. Submits the solution as an IPv8 message
4. Receives a response from the server (accepted or rejected)

For this, the code 

1. P2P Network Server Discovery
Overlay Joining: The script initializes an IPv8 instance and joins a specific target community identified by the Hex ID 2c1cc6e35ff484f99ebdfb6108477783c0102881.

Peer Discovery: It utilizes standard bootstrap definitions and a RandomWalk strategy to actively discover peers in the network.

Server Target Selection: A scheduled task (check_for_server) runs every 2.0 seconds, looping through discovered peers to match their public key against the 

hardcoded server public key:
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

# Assignment 2
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

2. Challenge 
The submisser changes across round acording to a fixed SUBMIT_ORDER = [1, 2, 3].
The submitter peer of that round requests a distinct round challenge via ChallengeRequestPayload.
Server Response: The server returns a cryptographic nonce and corresponding metadata through a ChallengeResponsePayload.
Internal Peer Broadcast: The submitter signs the nonce locally, then distributes the raw nonce directly to the other teammates using a NoncePayload message over the P2P overlay.
Signature Generation & Back-Propagation: Non-submitting teammates receive the NoncePayload and verify its round metadata.
Each teammate generates a local elliptic curve cryptographic signature using self.my_peer.key.signature(nonce).
They transmit this back to the round submitter using a SignatureSharePayload.

3. Signature Aggregation & Submission
Validation: The round submitter collects incoming signature shares and evaluates their cryptographic integrity using verify_share().
Bundle Submission: Once all three unique, valid signatures (Member 1, Member 2, and Member 3) are assembled inside the ActiveRound tracking object, the submitter wraps them in a SignatureBundlePayload and sends it to the server.
Round Progression: The server evaluates the bundle, issues a RoundResultPayload, and the group continues to a next round (updates current_round) until all three rounds complete successfully. Teammates stay in sync via RoundDonePayload notifications.

The code can be run by typing in the terminal:
python assignment2.py --key assignmentskey.pem --port 8091

Where assinmentskey.pem is the private key that is registered to the server in assignment 1.

# Assignment 3
The assignment is to:
You and your two teammates from Lab 2 build IPv8 nodes that together run a 3-node Proof-of-Work blockchain. Each member runs one node. Your nodes must mine blocks, propagate them, converge on a single chain, and answer queries from the Lab 3 server.
Once you register, the Lab 3 server joins your blockchain community, submits a test transaction, and walks every chain to check PoW, header linking, body commitment, and 3-way consistency. Your group passes the first time those checks all hold.

Our final solution uses the following key points:
Decentralized Mining: All team members run the mining loop concurrently to compete for block discovery.
Longest-Chain Rule: Automatic fork resolution and chain reorganizations happen when a peer discovers a longer valid path.
Transaction Gossiping: Transactions submitted to any node are flooded to all teammates, ensuring synchronized mempools across the network.
Reorg-Safe Mempool: Transactions from orphaned/abandoned blocks are automatically rescued and pushed back into the active mempool to ensure zero data loss.

To illustrate how it works, we use the following step by step  example:

## Step 1: Bootstrapping & Server Registration (Node 1 Startup)
Script Initialization: You run Node 1 with the --register and --test-mode flags. IPv8 starts up and instantiates two overlay networks locally: Lab3GlobalCommunity and Lab3BlockchainCommunity.
Server Discovery: Node 1 utilizes its built-in bootstrap definitions to discover the central assignment verification server (LAB3_SERVER_PUBLIC_KEY_HEX).
Registration Request: The lab3_global_loop task fires every 0.3 seconds. Node 1 notices it has found the verified server and transmits a RegisterPayload containing your Group ID and your local blockchain community ID.
Server Confirmation: The central server processes the registration and returns a RegistrationResponsePayload. Node 1 prints the message, successfully marks its global registration task as complete (self.done.set()), and shuts down the global community to save bandwidth.

## Step 2: Injecting and Gossiping the Test Transaction (Node 1)
Mock Generation: Simultaneously, the background lab3_consensus_loop runs every 1.0 second on Node 1. Because it is in --test-mode and sitting at the genesis block (Height 0), it triggers try_submit_transaction_test().
Transaction Signing: Node 1 creates a mock transaction payload using its own cryptographic keypair, signs the payload, and feeds it locally into on_submit_transaction_impl().
Mempool Insertion: Node 1 validates its own signature, registers the transaction in self.known_transactions, and drops it into self.mempool.
Standing By: Because Node 1 doesn't have any connected teammates yet, it updates its logs and waits.

## Step 3: Peer Discovery & Network Sync (Nodes 2 & 3 Startup)
Joining the Network: You turn on Node 2 and Node 3 without registration or test flags. They initialize their local Lab3BlockchainCommunity overlay networks.
P2P Peer Discovery: Utilizing the configured Strategy.RandomWalk and Strategy.EdgeWalk discovery routing strategies, the nodes discover one another's network addresses. Their local on_peer_added() methods fire, printing verification logs confirming that the teammates have bound together in the mesh.
Transaction Flooding (Gossip): The moment Node 1 registers Node 2 and Node 3 as active teammates, its internal transaction handler pushes the original test transaction payload across the network to them using ez_send().
Mempool Synchronization: Nodes 2 and 3 receive the payload via on_submit_transaction(). They verify Node 1's cryptographic signature, add it to their respective self.known_transactions dictionaries, and pop it into their local self.mempool queues.

## Step 4: The Competitive Mining Race
Race starts: The lab3_consensus_loop tasks on all three nodes fire simultaneously. Each node discovers that self.mempool is no longer empty.
Hashing work: Each node independently grabs the transaction from the pool, establishes a block header pointing to the GENESIS block's hash as prev_hash, and resets its nonce to 0 (since it is a new hash puzzle, that uses the prev-hash in it, the old nonce gives no information anymore, but you want to keep the number small because it it is defined to be 64 bits).
Cryptographic Execution: All nodes start looping through nonces via _mine_header(). They are racing to compute a SHA-256 hash that meets the target criteria: 22 leading zeros (DIFFICULTY = 22).

Now assume for example node 2 is the fastest:
Block Resolution: Node 2's CPU calculates a winning nonce first. Its _mine_header() loop breaks, creating a valid Block structure at Height 1.

## Step 5: Block Announcement & Longest-Chain Rule Adoption
Local Block Adoption: Node 2's CPU calculates a winning nonce first! It builds a valid Block at Height 1, updates its local self.tip_hash, clears its mempool, and uses _announce_block() to broadcast a BlockchainAnnounceBlockPayload to its teammates.
Handling Concurrent Blocks (The Fork): If Node 3 finishes mining its own version of a Height 1 block at the exact same time, it also broadcasts its block to the network.
The Tie-Breaker Rule: When Node 1 receives both competing Height 1 blocks, its code executes the core structural check in _adopt_block():
if block.height > self.current_height():
Whichever block Node 1 receives first (e.g., Node 2's block) will pass this check because $1 > 0$. Node 1 updates its tip to Node 2's block.When Node 1 receives the second block (Node 3's block), the rule evaluates $1 > 1$, which is False. Node 1 saves Node 3's block in its history (self.blocks_by_hash) but does not switch its active tip. The network is now temporarily split (forked).
Resolving the Tie via the Longest Chain: The tie is broken in the next mining round. All nodes keep competing. If Node 3 successfully mines a block at Height 2 on top of its own branch and broadcasts it, Node 1 receives it.
Chain Reorganization (Reorg): Node 1 verifies the new Height 2 block. It sees that its parent exists in history (Node 3's block) and evaluates the height rule: $2 > 1$ (True). Because Node 3's chain is now strictly longer, Node 1 drops Node 2's block as its active tip, calls _rebuild_canonical_index(), and shifts its entire canonical history over to Node 3's longer chain.
Mempool Rewinding: Immediately after switching to the longer chain, Node 1 runs _drop_confirmed_transactions(). It reviews the transactions in the newly accepted chain. Any transactions that were in the abandoned Node 2 block but are missing from Node 3's longer chain are safely rescued and put back into self.mempool so they can be mined in the next block.

