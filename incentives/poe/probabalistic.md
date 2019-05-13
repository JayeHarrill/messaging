

We envision a mix network using a sphinx-like packet format and loopix-like cover traffic, in which mix nodes are rewarded for participating.  We want some cover traffic to later be randomly selected in a lottery and deanonymized to provide a proof that nodes routed traffic correctly.  

We think selecting winning cover traffic packets should support strong anonymity properties, so long as those packets reveal nothing about users.  At the same time, we shall discover below that any control over winning cover traffic packets supports "malicious mining" in which packet creators bias rewards towards their own nodes.  We therefore design our protocol to reveal nothing about users, while requiring proofs that winning packets were constructed without biases, including biases produced by proof-of-work attacks.

# Design space

We first discuss the higher level aspects of the design space for such a protocol that do not depend overly on the cryptography.  We do currently assume a solid working knowledge of the mix network literature, at least for many motivational point, but may elaborate on the details later.

### Reward vs punishment

There are two possible outcomes to revealed packs, maybe a success results in rewards, or maybe rewards accrue in another way and failure results in punishment, or maybe both.  We think success should result in a reward because otherwise we need another system to distribute rewards with perhaps equally complex security concerns.  We do not yet know if penalties can be avoided, but we favor avoiding them if possible, if only for simplicity.  We think minor penalties like loss of rewards sound simpler than larger slashing-like penalties because avoiding staking makes participation easier and keeps the protocol simpler.  Also, slashing-like penalties complicates running nodes for competent people, who may lack wealth due to being cautious. 

There is however a "poor quality nodes" problem in networks like Tor and I2P, although I2P does not recognise it, in which poor quality nodes increase latency for the entire network.  We might not notice extras latency but we expect a similar "unreliable nodes" problem, and care about nodes' bandwidth, so users might benefit from established nodes for important roles like aggregation points, contact points, or even guards or other cross-over points.  We could represent reliability with stake directly, but we prefer to assess reliability directly, perhaps using stake from "reliability nominator" or "predictor" nodes who get slashed for poor enough behavior.  We might steal ideas for this from the availability game in polkadot.  In any case, we can leave assessing reliability as future work for after we have aggregation point code and applications that utilize it.

We therefore choose to reward based on successful packets for our immediate design goal, but leave assessing reliability to future work.

### Payment for only full routes vs partial routes

We shall pay only for payment packets that fully complete their route.  Importantly, this creates no anonymity problems because only cover traffic packets win the lottery and pay out.  

We could attempts to pay for good nodes in an initial segment of a partially completed route, but this creates many problems:  We think payments for partial routes is significantly more complex as discussed below under replay protection commitments.  Indeed we have not even fully answered the question of how one correctly identifies the node who dropped a packet and do not know if a solution exists.  If we have a stratified topology, then partial routes are unfair to good nodes who appear later in the topology, which creates nasty political problems.  Also, cycling the topology frequently would fiber the anonymity set.

### Relay vs user generated cover traffic

There is cover traffic generated by both relays and users in Loopix, so reward generating cover traffic could come from either or both.  Ideally both sounds best, but more complex. 

Relay generated cover traffic comes from all relays in Loopix, not just guard nodes or aggregation points for us.  We can enforce that nodes generated traffic correctly by threatening to void their rewards.  We do not envision slashing because we do not envision all nodes being staked.  We have no similar threats against users, well not unless they pay or get paid too, but if most traffic is user generated then we may encounter other reasons for preferring user generated cover traffic.

As a counterpoint, we worry that if only relays generate cover traffic then running bad virtual nodes that generate cover traffic and censor users becomes highly profitable.  We could stake and slash such nodes, but doing so fairly sounds hard, worse than the availability game.

We shall revisit this point soon after considering further design criteria, especially the route selection section below, but all cover traffic being eligible for the lottery sounds preferable, even if we do not implement that initially.  

### Social vs. cryptographic solutions

We shall discuss cryptographic approaches to route selection bias below.  If route selection were solved, then it's plausible that running more nodes is always the optimal strategy for small players, but large enough players eventually benefit from running machines that spam cover traffic.  In this model without route selection bias, any player spamming honest traffic should still benefit from more real users, so that they could spend money running nodes instead of fake users. 

In reality however, we cannot completely solve route selection because an adversary with enough nodes and enough fake users producing candidate routes could always choose not to expose routes unless they themselves win.  We might address this by making the winning traffic cover packets publicly verifiable somehow, which requires keeping winners off the queues that contain real messages, or more likely off users entirely.

There is a separate problem that any sufficiently large operator inherently compromises the network's anonymity goals.  Indeed, some adversaries would do this for reasons besides financial rewards.  

We believe this could only really be addressed by encouraging human interactions between relay operators, which deanonymize relay operators to one another and permit assessment of unusual behaviour.  These human interactions might help determine layers in a stratified network topology with relay operators who could vouch for one another landing in the same layer and ideally monopolising entire layers, so that such adversaries never see packets' entire routes. 

We note that Katzenpost currently takes an even more dramatically human centered approach by requiring that all nodes belong to quasi-activist organisations.  These organisations charge users for network access, or raise funds other ways, store their own users messages, and handle abuse manually.  We dislike this strategy because it prevents casual participation, does not provide many useful properties like per-user DoS protection, does not provide a distributed service, and imposes considerable complexity on operators.

We nevertheless believe that human social relationship must be prioritised over cryptographic solutions for route selection, etc., perhaps even the easy solutions based on Intel SGX and Arm TrustZone, because ultimately those relationships defend the anonymity.  We could perhaps develop an endorsement scheme among node operators, or key signing parties, but use a stratified topology with restricted routes that (a) minimizes packets forwarded between friendly nodes and (b) isolates all nodes not participating in endorsements into one or two layers.

We do expect automated route selection to increase in importance to counter financial misbehaviour though.


# Reward protocol

We imagine some user Alice has constructed a sphinx packet header with curve point P_i at hope R_i and that some fragment of the packet's path looks like
  .. -> R_0 - R_1 -> R_2 -> R_3 -> ..

## Relay replay commitments

We worry that Alice need not actually send her packet but merely generate it and then sometimes create payments without anyone doing any work.  Arguably this sounds non-problematic if Alice spends money, but without that assumption it sounds problematic.  We therefore make nodes commit to the packets they process before holding the lottery.

In future, we might penalize excessively large commitments from nodes that dropped many packets, which helps defend against running completely fake nodes that merely construct their replay databases.

### Delivery receipts  

Associated to P_i at R_i, we define a 128 bit value that R_{i+1} computes and sends back to R_i given by
  delivery_receipt = H(KEX(P,R_{i+1}) ++ "Reward Auth Out")
We could then shield delivery_receipt from R_{i+1} if R_i computes another value
  safer_delivery_receipt = AES(kex_dr, reward_auth_back)
where
  kex_dr = H(KEX(P,R_i) ++ "Delivery Receipt Safety")
If we were not using probabilistic payments, then Alice could encrypt a payment to R_i using delivery_receipt or R_{i-1} could do so using safer_delivery_receipt.

We recall that sphinx requires mix nodes store replay protection databases, which must live as long as packet headers, including SURBs.  We envision the payment protocol requiring that nodes produce some commitment to their replay protection database, and reveal secret information from those commitments.  So our replay protection records have the form (replay_hash, ...) where
  replay_hash = H(KEX(P,R_i) ++ "Replay Protection")
and ... is whatever secret information we require. 

We could store delivery_receipt or safer_delivery_receipt in this secret information, which R_i could learn only from the subsequent hop R_{i+1} or Alice.  If we store only this, then a packet appearing to die at hop R_{i+1} could mean R_{i+1} dropped the packet, or R_i and Alice together forged the record from R_{i+1}.  This seems okay if we only pay only for full successful routes because R_{i+1} must sign their own commit and reveal messages anyways.  

It's plausible R_{i+1} might do so falsely for profit, but we could add some outright fake queries that resulted in penalties.  We might prevent false accusations by ensuring the fake queries were generated later than the traffic.  If done correctly, this certainty permits quite severe penalties, although currently we do not expect that ordinary nodes have stake. 

### Deniability vs signatures

We could help support partial routes if R_i stores a signature provided by R_{i+1} 
  reward_sig = S(R_{i+1}, delivery_receipt)
If R_i signed his commitment to his replay protection database, then this signature proves that either the packet was forwarded or else that R_i and R_{i+1} colluded, avoiding the collusion between Alice and R_i case.  There are two issues with using a signature here:

First, if we do even a simple Schnorr signature here, then nodes must compute an extra public key operation r = g^k used in the signature.  R_{i+1} cannot improve performance by using r repeatedly so long as R_i sees and verifies the signature.  If R_{i+1} only commits to a signature, then conceivably R_{i+1} could delete some vital information when claiming a packet, thus preventing two signatures by the same r being exposed, so maybe r can be reused for dramatic computations savings.  We might want to prevent claiming too many packets in this way, but it makes the system fragile when relays restart, etc.

Second, we fear storing signatures stores evidence about all packet's routes.  In principle, we achieve some deniability here if we define k = H(sk_{i+1} ++ P_{i+1}) and r = k P_{i+1} and store only s = k - sk_{i+1} H(r ++ M) but immediately delete r rather than storing it with the signature.  In payment, r can be recomputed by R_{i+1} only after Alice reveals P_i and signature can then be verified as usual, but this sounds like several more protocol moves.  We might still report r to R_i to verify the signature.  R_i knows P_{i+1} but deletes it.

## Route selection for cover traffic

We want running nodes to always be the most economical method to acquire tokens.  In particular, we want nodes to generate their cover traffic honestly, neither biasing the route, nor sending excess cover traffic.  At the extreme, we're happy with proof-of-useful-work, perhaps best named proof-of-onion here.  We must not degenerate into proof-of-useless-work, ala bitcoin however. 

We face a hurdle that cover traffic senders could bias their cover traffic towards their own nodes, thereby favouring themselves when their own cover traffic packets win the lottery.

At first blush, there is a relatively straightforward sounding scheme for route selection, given some strong assumptions on timing and network view:  We consider some node Alice that generates cover traffic eligible to win the lottery.  Alice has a VRF key v to which she is committed.  Alice seeds her VRF with some representative for the current time to produce a verifiable CSPRNG, probably by using the output to seed ChaCha20.  Alice uses this CSPRNG to sample her route from the network view.  

We discuss several complications here and propose solutions.  These solutions have notable code complexity overhead, but they only arise when Alice's cover packet wins the payment lottery, giving them acceptable performance even for slow solutions.  We suggest initially addressing these concerns with three dirty Intel SGX and Arm TrustZone hacks, and requiring that users clocks not be too skewed, but even this could wait until the network sees significant adoption.
TODO: NOT ANY MORE

### Alice should be uniquely committed to her VRF key 

We cannot ask that intermediate relays verify Alice's VRF key when forwarding packets because they have trouble verifying many properties, like say uniqueness.  Also, the VRF key lives in a pairing based curve, which hurts performance if evaluated everywhere.  

Imagine first that relays generate the cover traffic that wins the payment lottery.  A relay Alice could run multiple virtual relays to obtain more VRF keys.  We could however employ stake here if either (a) not all relays need to generate cover loops or (b) all nodes were staked.  We dislike all nodes being staked, but all nodes do generate cover traffic in Loopix.  In any case, Alice could still run numerous minimally staked relays, so winning musst be tied to stake in this scenario.  

We might initially ask that nodes stake themselves according to their self assessment of their bandwidth and reliability, but this sounds unrealistic long-term.  We might design some scheme in which auditor nodes estimate relays bandwidth, reliability, and honesty, while nominator nodes provided stake for unstaked replays based on the auditors recommendations.  We consider this approach extremely complex so we do not attempt to answer questions like rewarding auditors.  We might discover that some auditor-like functionality to be unavoidable though, in which case self assessment might be an acceptable initial starting place.  

If we devise some punishment scheme, the we could evaluate if merely loosing of reward provides sufficient economic incentives when all relays generate cover traffic but without stake.  If so, then this provides one simple solution.  There are also intermediate schemes that simulate stake by making a relays' first few winning packets become its stake, but again assuming punishment.  

We think some solutions described below for user generated cover traffic winning also works for relay generated cover traffic winning. 

Imagine second that users generate cover traffic that wins the payment lottery.  An honest guard could enforce that a user Alice commits to her VRF key, but what comes first the guard selection or the VRF key commitment?  Alice could run her own guard or connect to many guards, if either gave her more VRF keys.  There are practical issues with Alice's VRF selecting her guard, like guards being down, so the route selection solution might not work at guards anyways.  I'd expect this creates hiccups in using a stratified topology with restricted routes.  We might simply publish user's IP addresses when their cover traffic wins, but this likely creates privacy problems. 

We could certify Alice's VRF key from another node using a blind signed certificate, except naively that node could issue an unlimited number of blind certificates.  We could employ a threshold blind signed certificate though.  I dislike the complexity here but if Alice is expected to pay for services, then a threshold blind signed certificate provides a convenient avenue for that payment.  We could even produce a zero-knowledge proof that she burned money on chain and exploit an existing threshold blind signature by validators.  We might certify Alice's VRF key using other unique resources like a phone number, again incurring complexity from threshold blind signed certificates.  We might ask if rerandomizable certificates like Coconut could play any role here: https://arxiv.org/pdf/1802.07344.pdf

An interestingly straightforward solution would likely be to certify VRF keys using the machine itself as a unique resource, via Intel SGX and Arm TrustZone.  These lack strong threat models but they suffice for preventing malicious mining.  If we run the VRF outside the trust boundary, then Alice could employ malware to mint VRF keys, but she could equally well create numerous phone number registrations with Android malware.  iPhones have no usable trusted computing system, so maybe this works only for relays, or maybe iPhone users can pay real money.

We foresee two realistic straightforward short-term solutions here:

 - Commerce:  Alice pays for her VRF key so only user generated cover traffic wins.
 - We certify relay VRF keys using Intel SGX with only relay generated cover traffic winning.
 ...

We should investigate if proof-of-stake blockchain tools provide another better solution, like Ouroboros limiting block outputs or whatever.  If we solve network view below via something like Blockmania then some multi-teared winning notion might circularly validate VRF keys for both relays and users.

### Alice should sample the full network fairly

Imagine first that Alice pays for registering her VRF key, but chooses network nodes however she likes.  If she controls relays in each layer, then she could sacrifice her anonymity to extract free services from relays owned by others.  We do not consider this an unpleasant but not necessarily catastrophic attack.  We might ensure Alice's nodes contribute network services, but this requires punishments.

If instead Alice pays nothing, then Alice could pick routes that pay her own nodes.  We thus want Alice's packet build algorithm to sample random nodes from the network fairly and that this fairness be verifiable.

Assume Alice has some full network view L created by a public consensus process.  We may assume this consensus process certifies the fairness of L.  We assume that L is public and updated infrequently.  If Alice samples L using her VRF, then her sample's fairness becomes verifiable.  We observe the update frequency for L interacts with Alice's cover traffic rate discussed below, but this seems acceptable if both use a common time period.

We expect this network view L eventually becomes the scaling bottle neck for any anonymity systems, which bodes poorly for users naively generating cover traffic that results in payments.  Right now, Tor is already stretching their consensus by distributing the full list of 6000 nodes to 2 million users.  We might distribute more frequently since relays' keys rotate.

We believe such an L could be distributed to relays by making our underlying blockchain-based payment scheme yield extensive network information.   We thus concede that only relay generated cover traffic winning the lottery sounds much easier than user generated cover traffic winning.

We might investigate Jeff's index-based encryption for network PKI compression ideas, but these require new approaches to Byzantine fault tolerant protocols. 

There is a larger research project around users securely using a partial network view L, likely obtained through gossip.  We discuss verifiably sampling such a partial network view, independently from the anonymity risks in obtaining and using one.  

Imagine Alice commits to her network view L when she periodically reregisters her VRF key and that doing so produces a random seed r not controlled by Alice.  If she reregisters her VRF key often enough, then her network view still grows, but not as quickly.  Alice now produces a zero-knowledge proof that she looked up each node correctly using VRF(r ++ _).  These zero-knowledge proofs could likely be Boneh-Boyen short signatures that the node occupied the ith position in her list, although not leaking the list size matters too.  See: https://infoscience.epfl.ch/record/128718/files/CCS08.pdf

We do not however know anything about the fairness of Alice's network view L itself, so this scheme does not achieve the desired assurances.  We consider mode solutions unsatisfactory:

 - There are threshold identity-base mix networks solve this PKI problem, but only with catastrophic security sacrifices. 
 - We might agree on partial network views L_i and distribute these packaged L_i, but now using some particular L_i leaks considerable information about Alice's network view, probably breaking any security assumptions for a gossip based scheme.  
 - Alice might sample another node's network view using PIR, but computational PIR is painfully slow and information theoretic PIR creates numerous problems too.

We also worry that Alice's commitment to her view of the network slows her network view growth rate, which creates problems for new nodes.  

We might imagine some verifiable gossiping algorithm, perhaps designed by simplifying some blockchain scheme like X-Blockmania: https://arxiv.org/pdf/1809.01620.pdf  We expect "accounts" to simply be node info records, like IP address public key, and expiration.  As nodes reference one another's hashes they must incorporate account updates, so censoring nodes should only results in the censors' admitted network view growing smaller.  We're not exactly solving the ultimate scalability problem here, but at least users obtain some broad attestation to the network views they download.

We could imagine eventually verifying X-Blockmania participation and account database correctness with a zkSNARK.  We must worry that Alice intentionally starts by talking only with corrupt nodes, so those nodes must produce proofs too, which likely means zkSNARKs arranged in a cycle of elliptic curves.  We only know such constructions with 80 bits of security right now.  

We might consider attesting to some such gossip protocol's correctness using Intel SGX and Arm TrustZone too, which solves the design chick and egg problem.  There is however considerable engineering work required by such an approach because these technologies remain fairly limited, and doing a network protocol sounds outrageous.  iPhones lack any sufficiently powerful trusted computing scheme, so actually this approach only makes sense if we gossip nodes even among relays but only relay generated cover traffic wins the lottery.  In fact, all relays would use Intel SGX avoiding any second implementation for Arm TrustZone. 

An alternative monetary scheme might mitigates malicious mining attacks by sacrificing fungibility:  Alice's winning packet credits each relay one coins denominated jointly by all relays along the packet's route, perhaps the relays even collectively sign without any blockchain.  All users prefer coins for relays they consider honest because auditors observe high interlinking.  If a relay is honest then we want it to happily exchange any two coins denominated by itself, although auditors may play some role here too.  Alice might engage in malicious mining but doing so need to yield coins with poor linkage outside her relays because other users and relays interact with her nodes less than she does.  Users might pay for relay services like storage or large packets using the relays own coins.  We consider alternative monetary scheme like this highly speculative.

In summery, we see three potentially ultra-scalable moon-shot long-range plans:

 - Index-based encryption for network PKI compression gives a full network view, or
 - Alternative economic models with non-fungible coins

We see realistic two short term solutions here:

 - Commerce:  We gossip node information but make Alice pay for her VRF keys and accept the free services attack or investigate punishments. 
 - Tor style:  We construct a full certified consensus network view which we distribute to relays and initially users.  We might only have relays generate cover traffic win the lottery, so that future iterations could distribute less to users.
 - Gossip:  We design a verifiable gossip inspired by X-Blockmania or whatever.  We might eventually strengthening the verifiablility using Intel SGX or cycles of elliptic curves.


### Alice must evaluate her VRF sparingly

We do not want Alice to create too many cover traffic packets by sampling her VRF across a massive number seeds, be they time slots or index or block numbers or whatever.  

There are "VRF stretching" schemes that restrict the time slots when Alice's VRF samples could possibly become valid, thus making her VRF choose packets timings.  We do believe these could be made efficient, but they incur undesirable code complexity, like rarely creating zkSNARKs.  See ./_removed_/Stretching_VRFs

Instead, we observe that, if lambda is the rate paramater for cover traffic, then Exp(lambda) has a rather small variance lambda^{-2}.  We therefore propose that Alice be permitted to sample lambda = 1/E[X] packets per unit time for the rate paramater, so precisely the expected number of packets.  Alice might send more packets but such packets cannot win the lottery.  

Alice could send almost the same packets through multiple guards, but they would all win together while only one pays out.

## Winning the cover traffic lottery

We should select cover traffic packets that win the lottery by first producing some collaborative random number with which all eligible cover traffic producers like Alice hash each cover traffic packet header.  If these score low enough then that packet pays out.  We do include the route information when hashing, but we build packets from a PRNG so it suffices to include only the initial seed of the PRNG not the full packet header.  

As discussed above, we want to seed this PRNG with a stretched VRF denoted VRF_i(epoch_seed) where epoch_seed is some well determined previous collaborative random number.
  H(collaborative_random ++ VRF_i(epoch_seed)) < epsilon.  
We ensure VRF keys are selected well in advance of the collaborative random number, so that packets cannot be maliciously mined by merely choosing VRF, aka reduced to proof-of-work.

We mentioned earlier that selecting winning cover traffic packets should strong anonymity properties, so long as those packets reveal nothing about users.  

As a further justification, we observe that Loopix substitutes real packets into one stream of cover traffic.  We need not take winners from this stream, even if we do choose winners from users, but if we do take packets from this stream, then users must skip revealing any real packets.  We believe incorporating VRF output for winning packets reveals nothing about other packets, maybe thanks only to computational Diffie-Hellman assumptions, but VRFs seemingly require more complex assumptions.  We think skipping packets make the scheme more sounds more friendly to users going offline too. 

