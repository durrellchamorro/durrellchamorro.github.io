---
layout: post
title: "Be Aware of App not Being Aware of Database Data"
date: 2015-08-17 14:40:25 -0700
comments: true
categories: rails
---
When you write a database backed app, you need to be aware that the app isn't always aware
of new data that recently hit the database. For example,<!--more--> say you have a
method like this that deals one card to a player or dealer:

```ruby
def deal_one_card(role, hand)
   card = Card.deal_one
   if role.class == Player
     card.player_id = current_player.id
     card.hand_id = hand.id
   else
     card.dealer_id = dealer.id
     card.hand_id = hand.id
   end
   card.save
 end
```

and you have a loop like this that will continue adding cards to the dealers hand until
the total of the dealer's hand sums to at least 17

```ruby
while total(dealer.hand.cards) < ApplicationController::DEALER_MINIMUM
  deal_one_card(dealer, dealer.hand)
end

```

As the ```deal_one_card``` method above is written the card will be saved in the
database with the proper attributes linking it to the dealer's hand,
however the rails app will not be aware of this change in the database while inside the loop
so when it totals the dealer's cards, the total will never increase. Therefore if the total was < 17
entering the loop, it will stay < 17 and the loop will go on infinitely.

In order to let the app know how the ```deal_one_card``` method changed the database
you must reload the hand like so:

```ruby
while total(dealer.hand.reload.cards) < ApplicationController::DEALER_MINIMUM
  deal_one_card(dealer, dealer.hand)
end

```
This was a nasty bug I had the toughest time figuring out and I'm very thankful to Brandon
at <a href="http://gotealeaf.com/">Tealeaf Academy</a> for figuring this one out and explaining it to me.

