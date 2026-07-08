# Open-world PvP is always-on and consequence-gated, not context-gated

Combat classes only matter if fighting can happen where players live. Two stances were on the
table. Context-gating (PvP off in the open world, live only inside arenas or declared wars)
makes the server safe but guts the combat identity: armor certifications, battle fatigue, and
the Pig Rider justice loop go dormant day-to-day, and a mastered style is cosplay outside a
scheduled event. Consequence-gating (PvP live everywhere, but aggression carries a cost) keeps
styles meaningful everywhere and leans on infrastructure the server already runs.

We chose consequence-gating. Open-world PvP is always live. Unprovoked aggression spikes
consent-weighted Patrol heat, which drives the existing warrant, arrest, and record pipeline
(Patrol golems and player Pig Riders feed it; Mehen holds the bans and records). Two carve-outs
sit on top: a Declared War (Mehen-mediated between groups) suppresses heat between belligerents
inside its scope, giving large-group conflict a sanctioned outlet; and opt-in duels or arenas
carry no heat, for sport and practice.

The trade-off we accept: this exposes players to open-world ganking, mitigated (not eliminated)
by the heat response. We take that over a safe server because the whole design (crafted-only
gear, item decay, an SWG-style economy) assumes conflict with consequences is the substrate,
not an occasional event.

Consequences flow through systems built for the Folia server (Patrol, Mehen). The combat layer
lands in backend Phase 2 and must be built heat-aware from the start: damage events carry the
context that feeds heat weighting. Revisit if Patrol heat cannot scale to always-on open-world
load, or if playtest shows ganking drives players off before the heat response bites.