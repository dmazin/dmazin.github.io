---
layout: post
title: "Bristol, Day Two"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2026-02-16
tags:
description: A day of goats, churches, and brutalism in Bristol.
---

<style>
figure {
  margin: 1.5em 0;
}
figure img {
  display: block;
  max-width: 100%;
}
figcaption {
  font-style: italic;
  color: #666;
  margin-top: 0.4em;
  font-size: 0.9em;
}
.carousel {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  gap: 0;
  -webkit-overflow-scrolling: touch;
  margin: 1.5em 0;
}
.carousel figure {
  flex: 0 0 100%;
  scroll-snap-align: start;
  margin: 0;
}
.carousel figure img {
  width: 100%;
  height: auto;
}
.carousel-wrapper {
  position: relative;
  margin: 1.5em 0;
}
.carousel-wrapper .carousel-btn {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(0,0,0,0.7);
  color: white;
  border: none;
  font-size: 2.5rem;
  line-height: 1;
  padding: 0.2em 0.5em;
  cursor: pointer;
  z-index: 1;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.3);
}
.carousel-wrapper .carousel-btn:hover {
  background: rgba(0,0,0,0.9);
}
.carousel-wrapper .carousel-prev { left: 12px; }
.carousel-wrapper .carousel-next { right: 12px; }
.carousel-dots {
  text-align: center;
  padding: 0.75em 0;
}
.carousel-dots .dot {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #ccc;
  margin: 0 5px;
  cursor: pointer;
  transition: background 0.2s;
}
.carousel-dots .dot.active {
  background: #333;
}
</style>

<script>
document.addEventListener('DOMContentLoaded', function() {
  document.querySelectorAll('.carousel-wrapper').forEach(function(wrapper) {
    var carousel = wrapper.querySelector('.carousel');
    var prev = wrapper.querySelector('.carousel-prev');
    var next = wrapper.querySelector('.carousel-next');
    var dots = wrapper.querySelectorAll('.dot');
    var figures = carousel.querySelectorAll('figure');
    var current = 0;

    function scrollTo(i) {
      if (i < 0 || i >= figures.length) return;
      current = i;
      figures[i].scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'start' });
      dots.forEach(function(d, j) { d.classList.toggle('active', j === i); });
    }

    prev.addEventListener('click', function() { scrollTo(current - 1); });
    next.addEventListener('click', function() { scrollTo(current + 1); });
    dots.forEach(function(d, i) { d.addEventListener('click', function() { scrollTo(i); }); });

    carousel.addEventListener('scroll', function() {
      var scrollLeft = carousel.scrollLeft;
      var width = carousel.offsetWidth;
      var newCurrent = Math.round(scrollLeft / width);
      if (newCurrent !== current) {
        current = newCurrent;
        dots.forEach(function(d, j) { d.classList.toggle('active', j === current); });
      }
    });
  });
});
</script>

<figure>
  <img src="/assets/images/bristol/general_bristol_pic_from_brandon_hill_--_use_first.jpg" alt="Bristol from Brandon Hill">
  <figcaption>Bristol from Brandon Hill</figcaption>
</figure>

Today was day two of my trip to Bristol.

The day started, unfortunately, at 4:30 AM, when I woke up and could not get myself back to sleep. My child has trained me to wake up at 6 AM, as that is when he wakes up. Any time I wake up before him feels like a criminal waste. Since he is in Florida, and I am not responsible for him at all, I ought to be waking up no earlier than 10. But, no, I rise very early as always. And so I writhed around in bed until 7:30 or so.

At 8 I ran down to an off-licence, not wearing a jacket or hat or gloves, just the one pair of (fashionable) trousers (that I stole from my wife) I bought and my huge Armor Lux sweater. I carried my phone in my hand and felt effortlessly casual. Sometimes you see a guy walking down the street, carrying his phone in his hand. I don't mean looking at it. I mean just carrying it. For some reason, to me that always seems like an important guy. Not because he needs easy access to his phone. But because he doesn't... like, have pockets? But I did have pockets! I don't know. I felt good. (What did I buy at the off-licence? Toothpaste. Behind the counter, there were a bunch of lighters, all of them covered in naked women.)

Back at the B&B, I had breakfast. It being a B&B, at 9 a very nice lady cooked me a bacon and egg sandwich. It was very good.

Heading back out, this time in my full winter gear, I headed down to an Oxfam bookshop I had noticed. It was a full two-story bookstore. Well curated: Tufte, Shock of the New, a book on Norman Foster. Redditors in generic UK forums love to complain about all the charity shops on high streets. Coming from the US, where we love thrift stores, this seems like a ridiculous complaint. Oxfam and Amnesty tend to be expertly curated. What do they want instead?

<figure>
  <img src="/assets/images/bristol/ofxam_exterior.jpg" alt="The Oxfam bookshop">
</figure>

When I entered, one of the workers was playing PJ Harvey. Another asked him if maybe he was playing too much PJ Harvey, to which I said, "Aw, keep PJ Harvey on!" but they had already switched it to classical. The worker who had complained said, "Ah, I was only asking a question. I don't know who PJ Harvey is. He must be young." to which I said, "I think PJ Harvey is older than you" (the worker looked in his 50s). Anyway, I went upstairs to browse a bit. But I did not get anything -- my backpack is already stuffed to the brim.

After Oxfam, I found myself wringing my hands about next actions. I was thinking of either walking along the River Avon or visiting the local botanical garden. I decided on walking the River Avon, which meant walking to the famous Clifton Suspension Bridge. I am staying at the top of Brandon Hill, so I snaked around the hill, vaguely in the direction of the bridge. It was here I noticed that, it being mid-February and the UK getting a bit more sunshine, the first flowers are blooming.

<div class="carousel-wrapper">
  <button class="carousel-btn carousel-prev" aria-label="Previous">&lsaquo;</button>
  <button class="carousel-btn carousel-next" aria-label="Next">&rsaquo;</button>
  <div class="carousel">
    <figure>
      <img src="/assets/images/bristol/new_growth_1.jpg" alt="Hello, new life!">
      <figcaption>Hello, new life!</figcaption>
    </figure>
    <figure>
      <img src="/assets/images/bristol/new_growth_2.jpg" alt="Hello, new life!">
      <figcaption>Hello, new life!</figcaption>
    </figure>
    <figure>
      <img src="/assets/images/bristol/new_growth_3.jpg" alt="Hello, new life!">
      <figcaption>Hello, new life!</figcaption>
    </figure>
  </div>
  <div class="carousel-dots">
    <span class="dot active"></span>
    <span class="dot"></span>
    <span class="dot"></span>
  </div>
</div>

Between Berkeley Square (where I'm staying) and the bridge is, I think, a charming neighborhood called Clifton. I have a trained eye for plaques, and am proud that I did not miss a small plaque on top of a litter bin: "This box is dedicated to Craig of Royal York Crescent, who spends many a restful moment hither." I immediately got excited, thinking this was the best kind of memorial -- one that commemorates a beloved local animal (the best, probably, on earth is in Kentish Town: "Boris the cat lived here 1985-1995"). I looked it up, though, and it appears that some local guy made it up because he thinks it's funny or something. I don't know. The story turns out to be a tad boring (that a made up man leans against a litter bin -- less charming than a cat who basks on it).

<figure>
  <img src="/assets/images/bristol/litterbox_plaque.jpg" alt="The Craig plaque">
</figure>

Eventually, I made it to the park that surrounds the suspension bridge, the Clifton Downs. Entering, a large sign explains that the Downs is a large park, and, importantly, promises goats. I incorrectly read that this is the only place in the UK with such goats, which in hindsight obviously cannot be true. Instead, the goats eat invasive scrub to help defend the many rare plants of the Avon Gorge (which the suspension bridge spans), which indeed only grow here.

<figure>
  <img src="/assets/images/bristol/picture_of_bench_--_use_it_in_the_bristol_downs.jpg" alt="What is this Rodin-ass bench?">
  <figcaption>What is this Rodin-ass bench?</figcaption>
</figure>

And then the bridge and gorge came into view. The gorge is breathtaking -- much deeper than I expected. I realized instantly that I would not be able to go down to the Avon trail, it being hundreds of feet down. But I already knew my goal: to find the goats.

Let's take a moment to discuss the bridge. The bridge is obviously the most iconic landmark in Bristol. When I get Instagram ads to visit Bristol, they show the bridge. It was designed, as seemingly half the objects in the UK, by Brunel. How did they build this bridge? I don't know. The gorge is so deep. Did they use cannons to shoot the cables across the gorge? How many people died building it? The answers to all these questions, I am sure, are on Wikipedia, for later.

<figure>
  <img src="/assets/images/bristol/bridge_shot.jpg" alt="Clifton Suspension Bridge">
  <figcaption>Obligatory, obviously.</figcaption>
</figure>

Right -- the goats. I had to walk through a municipal park for a while to get to the Gully, where the goats live. I walked and walked, not sure if I was even going to the Gully anymore, until I saw a bunch of camper vans. I decided to beeline across a huge grassy field, past the camper vans, and went off script entirely, going into some brush. There were trails inside. I started going downhill, not sure what I was even doing (hoping that I was not heading towards a cliff edge), until I saw a gate. On the gate was a plaque about goats -- great. Also on the gate was a sign begging people to leash their dogs; apparently two goats and one dog had already died. I love these sorts of signs that tell a (sordid) story.

<figure>
  <img src="/assets/images/bristol/public_notice_about_goats.jpg" alt="The goat-and-dog sign">
</figure>

Past the gate, I continued precariously going downhill, using my hands for balance. I suddenly understood why some people's asses were muddy. But I did not slip. I started looking for the goats -- it was very windy -- and -- in the distance, there they were!

<figure>
  <img src="/assets/images/bristol/goats.jpg" alt="The goats of the Avon Gorge">
</figure>

<figure>
  <img src="/assets/images/bristol/goat_closeup.jpg" alt="Goat closeup">
</figure>

And so I had seen the goats. In my life, I have seen animals in unexpected places at least three times now: the lions on the side of the road (really) in the hills of Pasadena, California; the zebras on the side of the road (really) near Hearst Castle, also California; and these goats.

Climbing out of the Gully, I walked through a grassy field. It was extremely windy. You know the way grass blows behind a jet engine, on an airfield? Standing with my arms out, the wind was pushing hard enough to inflate my jacket. There were houses nearby. Did the people who bought them all know how windy it is here? I would not live on top of this windy hill. Everywhere in this area is incredibly windy.

I eventually made my way to the botanical garden. There was a whiteboard where someone had done a great job explaining what's up, botanically, in February. It is incredibly well written. It is such a joy to come across casually excellent writing, which I will quote in full:

> Winter Solstice has passed and the days are gradually getting longer. You will see the first Snowdrops are flowering along the drive as well as the pink Bergenia and purple, pink and white Hellebores.
>
> In the woodland area underneath the ancient Oak trees the ground is covered in a carpet of Crocus tommasinianus, the woodland crocus. Snowdrops and teeny, tiny Cyclamen are scattered amongst them as well as wild daffodils, Narcissus pseudonarcissus.
>
> Follow your nose to Phylogeny where the strong, sweet perfume from the pretty pink flowers of Daphne bholua 'Jacqueline Postill' wafts across the garden.
>
> Right behind you is another highly scented plant, Sarcococca confusa, which is covered in tiny creamy coloured flowers where some of the Orchids are flowering in the Sub Tropical House and you can see some amazing plants from all over the world.
>
> Enjoy your visit!

So those must be all the budding flowers I saw on Brandon Hill.

At the botanical garden, unfortunately, I was starving to the point of loss of concentration. I walked quickly through the outside areas, where much was dead or hibernating, to the glasshouses, which were full of life.

<figure>
  <img src="/assets/images/bristol/glasshouse_outside.jpg" alt="The glasshouse">
</figure>

Inside, the glasshouse had that glasshouse smell. You know the one I'm talking about? I remember the first time I smelt it, at a botany camp I took as a kid in Russia. At the end of the camp, the instructor gave me some sort of hybrid lemon plant, which I cherished for years.

<figure>
  <img src="/assets/images/bristol/glasshouse_inside.jpg" alt="Inside the glasshouse">
</figure>

<figure>
  <img src="/assets/images/bristol/glasshouse_detail.jpg" alt="Glasshouse detail">
</figure>

The glasshouse had lots of Californian stuff. I saw succulents which looked like the kind that grow all over the dunes of Southern California. They grow in huge amounts, covering huge areas. You have to walk through special paths to avoid them. Here, they were tiny, in tiny pots.

Also in the glasshouse were palms. I say this mostly because recently I learned just how much the Victorians loved palms. The Palm House was the primary attraction of Kew Gardens for more than a century, I think. In California, the palms all have some sort of virus and I once read they will be dead in a few decades. The palms of the British glasshouses will surely outlive them, some already being extremely old.

Anyway, I was starving, to the point where I ordered an Uber. It was hard not to spend £10 on a 10 minute ride considering the alternative was an hour-long bus.

My destination was Chilli Daddy, which I had found looking for Bristol's best Szechuan. Maybe it is Bristol's best, but it is certainly not good. The beef had little flavor, and the broth had little flavor. There was none of that tingling you expect. You order from an iPad and then a loudspeaker screams your number at you. No soul whatsoever. The presentation was good, though.

<figure>
  <img src="/assets/images/bristol/chilli_daddy_food.jpg" alt="Chilli Daddy">
</figure>

Last night, I had ordered the same dish at a similar restaurant. It was late at night. A very charming teenager asked me for it. They had beef stew of Malaysian kind, and of two other countries I forget. I asked the kid which one -- Malaysian, for sure, he said. And not just because he was from Malaysia, he added. He asked me how spicy I wanted it -- I said maximally spicy. Oh, we have a competitor. I'll tell the chef, he said. It was not spicy at all -- I don't think it's because I am a badass, I think they just forgot.

Sitting at Chilli Daddy, I read the Wikipedia article on the Blitz of Bristol. The Lord Mayor of Bristol, Alderman Thomas Underwood, described the effect of the raids: "The City of Churches had in one night become the city of ruins." One of the casualties of the bombing was Bristol's tram system. Apparently, in the churchyard of St Mary Redcliffe, you can find a memorial to the tram system. Being a transit nerd, and this in general being something of a transit nerd trip (tomorrow I'm riding the cross-country route to York, where I will obviously visit the railway museum), I knew I had to go. The internet said that St Mary Redcliffe was beautiful, in fact almost overshadowing the actual cathedral. I was also promised a gravestone commemorating the church cat.

<figure>
  <img src="/assets/images/bristol/redcliffe_chhurch.jpg" alt="St Mary Redcliffe">
  <figcaption>St Mary Redcliffe</figcaption>
</figure>

The church is indeed huge, though situated sadly off a huge roundabout. Outside, a sign apologized about staff shortages. Inside, indeed there were two ladies. Just two ladies, in a giant church. Are there any tours today, I asked. No? The lady said. She offered me a guide. But I already knew what I was after.

The church was as empty of visitors as it was of staff. But inside I saw a wonderful display.

One was a display marked "JOURNEY INTO SCIENCE: THE ST MARY REDCLIFFE CHAOTIC PENDULUM".

I can't get the video to work. So we will have to do with words. It is a pendulum which, through creative pumping of water, oscillates in an unpredictable way.

The full display is excellently written, and so I am quoting it in full:

> Water, which is recycled, slowly flows into the centre of the cross beam, which tips to let it out.
>
> But which way will it tip? What is remarkable is that with all the science in the world, no one can predict exactly how it will be moving a minute from now.
>
> This is the way the world is. In this simple machine, you are looking at a new frontier in our understanding of the world. Scientists call it chaos.
>
> Some people look to science for certainties on which to base their lives. Increasingly we realise our knowledge can never provide certainty, even for this simple machine. The world is a more wonderful and a more surprising place than we could have imagined.

I often think about John Berger saying that museum plaques almost feel like their goal is to obfuscate, so it is inspiring when the words on a plaque feel true, and from the heart: "This is the way the world is."

As the sign says, the display was devised by Dr Eric Albone. I wonder if he wrote these powerful words?

<figure>
  <img src="/assets/images/bristol/need_to_rotate._for_redcliffe_church.jpg" alt="What are they getting at here? I am wearing trousers already.">
  <figcaption>What are they getting at here? I am wearing trousers already.</figcaption>
</figure>

After wandering around for a while, lighting a votive candle for my dad and my wife's grandpa and paying respects to the giant organ, I found myself outside. I wandered around aimlessly a while. There were pieces of church scattered all around.

There was a memorial garden. A pot of flowers had fallen over; I righted it. I am moved whenever I come to a memorial, a memorial actively tended by someone whose loss is sharp. It's not just because I understand, but because it feels like the genuine way to engage: to participate, to spare a thought.

And then I found the tram memorial. Right there, out of the ground, a piece of rail sticking up.

<figure>
  <img src="/assets/images/bristol/tram_memorial_redcliffe.jpg" alt="The tram memorial at St Mary Redcliffe">
</figure>

But, frankly, I moved quickly onto the grave of the cat. I wanted to see the grave of the cat.

<figure>
  <img src="/assets/images/bristol/the_church_cat.jpg" alt="The church cat's grave">
</figure>

There was a flower by it, I don't know if on purpose or coincidence. It was the sort of flower recently blooming. I made the flower look pretty. Christianity has a famously warm relationship with cats; monasteries had them, anchorites and anchoresses had them, and many churches and cathedrals still have them (shout out to the Southwark Cathedral's cat!).

I realized that I was near Bristol Central Library and decided to pay a visit. The study room was supposedly pretty. Immediately inside, I saw the notice board. I am showing it here mostly to illustrate how deeply library staff care about the unhoused. They come to the same conclusion that I do: when you come across a person who needs help, engaging in a small way -- a brief conversation, £10 to help them get whatever they need -- can make a huge difference. It will at the least change their day.

<figure>
  <img src="/assets/images/bristol/centra_lib_expterior.jpg" alt="Bristol Central Library">
  <figcaption>Bristol Central Library</figcaption>
</figure>

<figure>
  <img src="/assets/images/bristol/central_lib_interior.jpg" alt="The notice board">
</figure>

The reading room itself was fine. However, I loved these book lifts, which were unfortunately dysfunctional.

<figure>
  <img src="/assets/images/bristol/book_lift_central_lib.jpg" alt="Book lifts">
</figure>

<figure>
  <img src="/assets/images/bristol/library_interior_central_lib.jpg" alt="The reading room">
  <figcaption>This is screaming to have one of those circular Cray supercomputers installed</figcaption>
</figure>

Leaving the library, I realized I was very close to the actual cathedral, so figured I may as well pay a visit. I am very glad I did. I do not know if anyone actually gives this advice, but do not take any advice that says to visit St Mary Redcliffe instead of the cathedral. The cathedral is full of people (and staff), full of interesting history. It is a working cathedral, and feels alive (unlike St Mary Redcliffe).

<figure>
  <img src="/assets/images/bristol/cathedral_how_many_staff.jpg" alt="Cathedral interior">
</figure>

<figure>
  <img src="/assets/images/bristol/cathedral_interior_about_how_many_staff.jpg" alt="Cathedral interior">
</figure>

As interesting as the interior of the church is (which apparently earned Pevsner's approval), this post is getting way longer than I expected it to, so let us focus on the cloistered garden, which was delightful.

As you enter the garden, you are presented with an informational sign, of the "ladies from clergy wives and members of our congregation" efforts to make the garden "a unique & unusual secret garden -- a hidden gem in the city centre." The sign adds, vaguely: "The weeping cherry [...] was planted by a Japanese lady who converted to Christianity while living in Bristol."

<figure>
  <img src="/assets/images/bristol/gachtral_cgarden_exterior_shot_main.jpg" alt="The cathedral garden">
</figure>

<figure>
  <img src="/assets/images/bristol/cathedral_garden_exferior_shot.jpg" alt="Cathedral garden">
</figure>

<figure>
  <img src="/assets/images/bristol/cathedral_garden_label.jpg" alt="The garden sign">
</figure>

Wandering around the garden, I heard a blackbird (I think?) singing from inside a bush. I knelt down and tried to take a video of it. The blackbird stopped singing. We stared at each other for literal minutes. I guess it's not surprising it stopped singing, because there was a guy sitting there staring at the bird.

<figure>
  <img src="/assets/images/bristol/blackbird.jpg" alt="The blackbird">
</figure>

And so I continued my tour. I had one more destination for the day.

Bristol was, if I recall correctly, a city much like Bath: full of beautiful Bath stone buildings. But heavy bombing in WW2 destroyed much of the city. Much of it was rebuilt after the war, and not necessarily for the better.

One of the much maligned buildings in Bristol is the brutalist Arts and Social Sciences Library at the University of Bristol. This I had to see. In Berkeley, the architecture building, Wurster Hall, has a very bad reputation. I think this is for a very bad reason. Wurster Hall -- forgive me for going off -- is a gorgeous building, and while it is incredibly cramped and low-ceilinged inside, has my favorite library in the world, where sitting inside you can see people walking and working up in the mezzanine.

<figure>
  <img src="/assets/images/bristol/wurster_exterior.jpg" alt="Wurster Hall, UC Berkeley">
  <figcaption>Wurster Hall, UC Berkeley</figcaption>
</figure>

<figure>
  <img src="/assets/images/bristol/wurster_library.jpg" alt="The Wurster library">
  <figcaption>The Wurster library</figcaption>
</figure>

Compare this to the actually horrendous math building in Berkeley, Evans Hall, which had to be painted a special shade to help it blend into the hillside, it being truly ugly.

<figure>
  <img src="/assets/images/bristol/evans_hall.jpg" alt="Evans Hall, UC Berkeley">
  <figcaption>Evans Hall, UC Berkeley</figcaption>
</figure>

So, the Bristol university building -- was it the former (wrongly hated) or latter (rightly) kind? As it came into view, it became obvious that it's the former.

<figure>
  <img src="/assets/images/bristol/ugly_lib_supposedly_main_exterior.jpg" alt="The Arts and Social Sciences Library">
  <figcaption>The Arts and Social Sciences Library</figcaption>
</figure>

While it undeniably looks like a fortress, this is a genuine architect's building with personality. Each slanted window has someone studying behind it.

<figure>
  <img src="/assets/images/bristol/ugly_lib_closeup_1.jpg" alt="Slanted windows">
</figure>

<figure>
  <img src="/assets/images/bristol/ugly_lib_alcove_cloup.jpg" alt="A glass bay">
  <figcaption>You're telling me people hate this?</figcaption>
</figure>

Each corner has one of these glass bays with someone studying in it.

I did not go inside, so maybe it's very bad in there.

Anyway, time to end this post, which turned out to be very long. I end it with two things.

Look at this sign. How do I get this cutout of a child holding "hey fucker, do not park here" signs?

<figure>
  <img src="/assets/images/bristol/do_not_park_here.jpg" alt="Do not park here">
</figure>

And, in Berkeley Square, these tiny little niches for your trash cans. I wonder what used to go in there!

<figure>
  <img src="/assets/images/bristol/trahs_alcove.jpg" alt="Trash niche">
</figure>
