```pikchr
# DEFAULTS

$step   = 0.3
linewid = $step
lineht  = $step
boxwid  = 1.2
boxht   = 0.4

define proposer    { box thin fill 0xBFBDFE $1 big rad 10px dashed }
define psc         { box thin fill 0xFEFFBF $1 big }
define implementor { box thin fill 0xFFBDFF $1 big rad 10px }
 
# MAIN PROCESS

# proposer

down
proposer( "cool idea" )
arrow
proposer( "take it to p5p" )
right
arrow
proposer( "popular" big "support?" )
arrow "no" big below
proposer( "give up" )
down
arrow from 2nd last box.s
proposer( "draft proposal" )
arrow <->

# psc

Review: psc( "PSC review" )

arrow from 2nd last box.w \
    left \
    then down until even with last box \
    then to last box.w <-

arrow from last box.s
psc( "Exploratory" )
arrow
text \
  "idea subjected to review" small bold rjust \
  "PSC must approve or reject" small bold rjust \
  at last arrow.w - ($step / 4,0)

# implementor

implementor( "Implementing" ) with .n at last arrow.s
arrow
text \
  "implemented by volunteers" small bold rjust \
  "PSC must approve or reject" small bold rjust \
  at last arrow.w - ($step / 4,0)

arrow from Implementing.s
implementor( "Testing" )
arrow
text \
  "implementation tested by volunteers" small bold rjust \
  "PSC must approve or reject" small bold rjust \
  at last arrow.w - ($step / 4,0)

box rad 20px fill 0xBDFFBE "Accepted" big with .n at last arrow.s

# REJECT

right
arrow 200% from Implementing.e
RoE: [
  down
  box thin "Rejected" big
  box thin "or" big italic ht 50%
  box thin "Expired" big
] with .w at last arrow.e

arrow from Review.s + ( $step, 0 ) to RoE.n - ( $step, 0 )
arrow from Exploratory.s + ( $step, 0 ) to RoE.w
arrow from Testing.n     + ( $step, 0 ) to RoE.w

# LEGEND

[ down
  boxwid *= 7/8
  boxht  *= 7/8
  WHO: box thin thin "who is responsible?" small fill white fit
  move $step/4
  proposer( "proposer" )
  move $step/4
  psc( "PSC" )
  move $step/4
  implementor( "implementor" big "& PSC" )
  box thin width last box.wid + $step height last box.ht * 4.25 \
    with .n at WHO.c behind WHO
] with .nw at 6th last box.ne + ( linewid * 3.5, $step / 2 ) 
```
