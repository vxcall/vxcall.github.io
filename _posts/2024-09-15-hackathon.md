---
title: "We Won 1st Place at Hackathon!"
date: 2024-09-15 14:06:00 +0900
categories: [web, backend]
tags: [rust]
media_subpath: /assets/img/posts/hackathon/
image:
  path: header.png
  lqip: header.svg
  alt: header
---

## Introduction

Victory is ours! Today is the day we --- Team Artizans got the 1st place in a hackathon.

September 15th 2024, marks the demo presentation day for the Revenue Cat hackathon in Tokyo, where I'm participating as a backend engineer.
I initially joined alone but teamed up with 5 amazing individuals on the first day of the hackathon.

Our challenge was to create a subscription-based app and launch it on the app store within a two-week timeframe.
That sounds crazy isn't it?:) considering that the initial iOS app store review typically takes about a week.
We needed to develop at lightning speed, ideally completing the project within a week.

This experience was not only thrilling but also an intensive training in building a backend from scratch under extreme time constraints. In this post, I'll share our journey, the challenges we faced, and how we overcame them to emerge victorious.

## Our Project

We developed Artomo, an app designed to make art more accessible to people who aren't typically interested in or exposed to it. Our primary goal was to break down the barriers that often make art seem intimidating or exclusive, and to introduce a wider audience to the world of galleries and exhibitions.

Key Features:

1. Personalized Preference Test: Users start with a quick, engaging test where they select preferences for colors, textures, and sounds.
2. AI-Driven Recommendations: Based on the user's choices, our AI system recommends specific galleries from our extensive database.
3. Custom Route Creation: The app generates personalized routes for users to explore their recommended galleries efficiently.
4. Notifications: As users approach a gallery, the app sends push notifications with intriguing information, such as viewing tips, messages from the gallery, or historical facts.

By focusing on accessibility and personalization, Artomo aims to spark interest in art among those who might not typically visit galleries.
Our solution goes beyond simply mapping art spaces; it creates a bridge between individual preferences and the vast, often overwhelming world of art, making the experience more approachable and enjoyable for everyone.

## Challenges and Solutions

### 1. Absence of a Designer in the Initial Development Phase

#### Challenge
For the first 4-5 days of the project, our designer was in Sweden and unable to collaborate with the team. Moreover, apparently he's calling himself a service-designer. At this point we had no idea what kind of design he's capable of.

#### Solution
1. Our project leader created wireframes to guide initial development.
   ![wireframe](wireframe.png)
   _Figure 1: Wireframe created by the project leader_

2. We implemented parallel development:
   - Frontend team: Worked on design-independent parts
   - Backend team: Started developing necessary endpoints

3. We urgently recruited an external freelance designer through team members' networks. We've got it work and she produced a cool design for us.
    ![design](design.png)
    _Figure 3: Final app design_

#### Lessons Learned
- The importance of flexible development plans to handle unexpected situations
- The value of diverse skill sets and networks within the team

### 2. Backend Development Challenges and Iterations

#### Initial Approach and Its Pitfalls

We initially built our infrastructure on Amazon Web Services (AWS), using ECS Fargate to run our API server developed with Rust and the actix-web framework. However, we soon discovered a critical flaw: while AWS Fargate is designed to be serverless, our implementation prevented it from entering a dormant state when idle. The actix-web server continuously listened for HTTP requests, which would have resulted in significant costs if left running 24/7.

#### Architectural Redesign

After thorough research, we decided to pivot from Fargate to AWS Lambda for our web server needs. This shift offered substantial benefits in terms of cost efficiency, as Lambda's true serverless nature allows it to scale down to zero when not in use.

New Technology Stack:
- Language: Rust (unchanged)
- Framework: cargo-lambda
- Infrastructure: AWS Lambda, API Gateway

Improvements:
1. Full adoption of serverless architecture
   - Each endpoint implemented as an individual Lambda function
2. Enhanced cost efficiency
   - Flexible scaling based on request volume

Infrastructure Configuration:
![er graph](er_graph.png)
_Figure 2: Improved infrastructure configuration (VPC and subnet details omitted)_

#### Lessons Learned
- The proper utilization of serverless architecture
- The importance of initial design and the value of bold pivots when necessary

### 3. Inter-team Communication Challenges

#### Challenge
We struggled with effective communication between the frontend and backend development teams.

#### Solutions and Ideas for Improvement
1. Regular all-hands meetings for progress sharing and early issue detection
2. Utilization of common documentation tools for sharing API specifications and data models
3. Introduction of pair programming sessions to promote mutual understanding

#### Lessons Learned
- The importance of proactive efforts to break down barriers between teams
- The need to focus on human factors, not just technical challenges

## Frontend Development

Although I'm not a frontend developer, I can attest to the challenges of maintaining proper communication between backend and frontend teams, especially in a non-job setting where daily catch-up meetings weren't feasible.

Our final design evolved significantly from the initial wireframe:

## Conclusion

![footer](footer.jpg)
_Figure 2: Improved infrastructure configuration (VPC and subnet details omitted)_

Legend has it that technology itself isn't the purpose, but rather a means to accomplish what you want. I believe our victory stemmed not only from our technical skills but also from our problem-solving abilities, diligent data collection, and the days we spent visiting galleries while conceptualizing the service.

In the end, it's not just about the technology; all the efforts we made count. This hackathon experience taught us valuable lessons in teamwork, adaptability, and the importance of understanding the problem we're solving beyond just the technical aspects.

For future projects, we'll pay special attention to:
1. Creating flexible initial plans
2. Considering long-term perspectives in technology selection
3. Continuously improving inter-team communication

By applying these lessons, we aim to achieve even more efficient and innovative project management in the future.

Thank you, Team Artizans, for this incredible journey and victory!