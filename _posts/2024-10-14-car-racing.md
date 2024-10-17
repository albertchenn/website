---
layout: post
author: albert chen
tags: [computer vision, hackathon]
---

## The Event

Entering my first ever hackathon, we had no expectations on what was going to happen. While we had some rough planning done before the event, we were unsure how the entire project workflow was going to be like, and what the end product exactly looked like. However, this uncertainty may have been exactly why we were so successful.

### The Idea

Originally, taking inspiration from other projects, we thought of creating a top down driving simulator through the city of Detroit.
We were planning on going through the automotive track, and had some small ideas on how we were going to complete this task.
However, we left our ideas open for what libraries we were going to use, and how exactly everything was going to look.

### The first day...

We had planned out our team before the hackathon, and I decided to work with [PJ](https://www.linkedin.com/in/peter-kim-34719126b/), [Dakota](https://www.linkedin.com/in/dakota-hanks-673310223/), and [Safwan](https://www.linkedin.com/in/safwan-khayer-a127a326b/).
The night before, I made sure to get a good night's rest, as I was sure I was not going to get one the following night.
Me and PJ got there at 10:15, about 15 minutes after the door opened and we explored the Career Fair, and tried to resist the urge to start working on the project.
Eventually, we were able to make it to the presentation, and after lunch we got working.

### Working

Immediately, we found a good spot to work and set up shop to start grinding through the project.
We first cleared up any misconceptions we had, and established and ironed out exactly what our goals were, 
and what we planned to do if we had more/less time than anticipated.

While we made sure to take breaks if we felt tired or burnt out, most of our time was spent working and collaborating,
making sure that progress was being made constantly.

### Presentation!

Approaching Demoing time, our project was nearing completion. We had the luxury to spend the time to attend a "Demoing for Dummies" workshop,
and learned valuable information on how we should present our project. We created slideshows and ran through practice presentations to make sure
we were ready to present. We felt as if we really knew our project, so we didn't feel the need to hammer down any details or questions.

Our demos went well! Most of the judges seemed to enjoy our game, and suggested some ideas that could allow
our application to have bigger impacts than just a fun game, which I'll talk about later. What really hit me most
was the effect it had on the other hackers, whom often came to check out our project and play the game for a bit,
often explaining how the game was "a lot more fun than expected". 

### Takeaways from the event

While we didn't end up winning anything, I felt as if I won. The event felt very successful, and really opened my eyes
to how far collaboration takes you. I wouldn't have been able to finish even a fifth of the project in double the time,
and it really emphasizes how exponential teammwork and collaboration is, in terms of efficiency. 

Also, I learned to branch out and talk to people. At the hackathon, there were so many people from different
works and aspects of life, and it was interesting to demo other projects and meet so many people. At the end of the day,
I didn't care about the physical outcome of the hackathon, instead I'll remember the people I met and the memories I made.

## The Project

I'll begin to give a deeper dive of the actual project now, so you've been warned.

### Computer Vision

For the computer vision, we originally chose to work with OpenCV.js, but unfortunately we realized
that it simply wasn't as fast as we'd like. After an hour of troubleshooting, we came across
Google's Mediapipe model, and it's hand gesture recognition. It seemed to be the exact functionality that
we wanted, so we decided to go through with it.

The gesture recognizer recognizes 20 different landmarks on the hand, but when your hands are in the driving position,
there's only 8 points that are really necessary. To account for this, every time we got the landmarks from the model,
we truncated the array of landmarks to the 8 points that we wanted.

![hand landmarks](assets/images/car-racing-images/hand-landmarks.png)

For the game, we wanted to extract two values from the hands: Relative Distance from Camera, and Angle of Hands.

For the relative distance we decided to follow this logic:
1. Determine area of each hand (truncated array) through the [Shoelace Formula](https://en.wikipedia.org/wiki/Shoelace_formula).
2. Average the two areas.
3. Use the following relationship to estimate distance. $$ d = \frac{k}{\sqrt{A}} $$
4. Subtract the distance from the zero'd distance (will explain later)

```javascript
// Shoelace
export function calculateArea(points) {
    let area = 0;
    for (let i = 0; i < points.length; i++) {
        let j = (i + 1) % points.length;
        area += points[i].x * points[j].y;
        area -= points[j].x * points[i].y;
    }
    return Math.abs(area / 2);
} 
// Use relationship
export function determineDistance(areas) {
    let averageArea = areas.reduce((a, b) => a + b, 0) / areas.length; 

    let distance = VELOCITY_MULTIPLIER / Math.sqrt(averageArea);

    return distance - zeroDistance;
}
```

To determine the angle of the hands we followed these steps:
1. Average the hand (truncated) to determine the center
2. Find the line through the two centers
3. Calculate the angle with `atan2()`

```javascript
export function calculateAngle(centers){
    let x = centers[0].x - centers[1].x;
    let y = centers[0].y - centers[1].y;
    let angle = Math.atan2(y, x);

    // cushion to make straight driving easier
    if (angle < ANGLE_THRESHOLD && angle > -ANGLE_THRESHOLD) { 
        return 0;
    }

    return angle;
}
```

We decided that it would be convenient for the user to be able to zero the relative distance from the camera. Therefore, we set the "Driving Mode" to a Thumbs Up on both hands, and "Zeroing Mode" to a not Thumbs Up (thumb on fist). 

To Zero the distance, we simply use this code block to reset the zero distance.

```javascript
export function setZeroDistance(areas) {
    zeroDistance = 0;

    zeroDistance = determineDistance(areas);
}
```

### The Game

One of the reasons we opted for a top-down racing simulator was because driving can be realistic when you simply just move the background sprite, so that the car can be stationary, and just rotate with the background.

Moving was fairly trivial, as we just needed to update the background's movement and the car's rotation. 

To rotate, we made sure it was within the bounds $$ (0, 2\pi) $$ by adding a check, and simply setting the car's rotation as the opposite of the target angle (from hands).

```javascript
function rotate(targetAngle) {
    if (targetAngle > Math.PI * 2) {
        targetAngle -= Math.PI * 2;
    }

    car.rotation = -targetAngle;
}
```

To move the background, we simply calculated the velocity vector's components based on the target velocity(from hands) and target angle(also from hands). 

```javascript
function move(targetVelocity) {
    // make zero velocity easier
    if (targetVelocity < VELOCITY_CUSHION && targetVelocity > -VELOCITY_CUSHION) {
        targetVelocity = 0;
    }

    let debuggingCode = document.getElementById('debuggingCode');
    debuggingCode.innerHTML = `Speed: ${Math.round(getCarVelocity())} m/s<br>Turning Angle: ${Math.round(getCarAngle() * 57.2958)} degrees<br>Car Angle: ${Math.round(car.angle)} degrees<br>Offroad?: ${offRoad}<br>Checkpoint: ${checkpoint + 1}`;
    
    // vector math to move the car
    background.x += Math.sin(targetAngle) * targetVelocity;
    background.y += Math.cos(targetAngle) * targetVelocity;
}
```

While the code seems relatively simple, it took a little longer than we expected to get here. Maybe your mind isn't that sharp after coding for 6 hours straight.

### File Structure

Keeping a clean, neat project in both the Git repository and Project filespace was a priority for us, and it came to save us near the end. First, we create a `constants.js` file, keeping all tuneable constants in one area, saving lots of time searching and troubleshooting. This also came in handy later on, when we added a settings page, allowing us to simply tweak the constants file.

Additionally, we commited 130+ times, allowing us to backtrack and keep a solid record of not only our progress, but facilitate collaboration and allowing multiple people to work at once.

## Conclusion

### What's Next
The judges who came to see our projects offered very valuable insight on the future of this application. With their advice, we were able to determine what possible next steps could be:
- Car Driving for those who are physically disabled
- Use for robots in dangerous locations
- Intuitive Steering for various robots

Additionally, for the game, we've come up with various features that could happen in the future:
- Login and Leaderboard system
- Multi track selector/creator
- Complex Driving Mechanics
    - Drifting
    - More sophisticated acceleration and steering.

### The End
This hackathon has been helpful in so many different ways, from being able to network out socially to learning the specifics of web and game development. While this project has been a little out of my comfort zone, sometimes you have to be able to step out of your safety net to grow.

My old swim coach used to say, you won't know your limit until you've reached it. I haven't reached my limit yet, so I'm going to keep going.

### Thanks
I'd like to thank [Hack Dearborn](https://www.hackdearborn.org/) for making my first ever hackathon an incredible experience.

Thank you to my teammates [PJ](https://www.linkedin.com/in/peter-kim-34719126b/), [Dakota](https://www.linkedin.com/in/dakota-hanks-673310223/), and [Safwan](https://www.linkedin.com/in/safwan-khayer-a127a326b/) for not only providing immeasurable amount of help, but for making a 24 hour coding session not just bearable, but fun! I couldn't have asked for any better teammates.

If you've made it all the way here, thank you for reading! 
If you want to check out the devpost, it's at [https://devpost.com/software/hand-gesture-racing-game](https://devpost.com/software/hand-gesture-racing-game), and our project is hosted [here]( computer-vision-car-game-64b0183ceff3.herokuapp.com)

![group picture](assets/images/car-racing-images/group.jpg)
