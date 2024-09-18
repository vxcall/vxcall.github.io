---
title: "team Artizans, journey in a hackathon"
date: 2024-09-15 14:06:00 +0900
categories: [backend]
tags: [rust]
img_path: /assets/img/posts/hackathon
image:
  path: header.jpg
  lqip: /assets/img/posts/hackathon/header.svg
  alt: header
---

## Introduction
Today, September 15th, marks the demo presentation day for the Revenue Cat hackathon in Tokyo, where I'm participating as a backend engineer.
I initially joined alone but teamed up with 6 amazing individuals on the first day of the hackathon.
Our challenge was to create a subscription-based app and launch it on the app store within a two-week timeframe.

## Our Project: Artomo

We developed a gallery assistant app to address the challenge of finding and exploring interesting art galleries in the city.
Many excellent galleries blend into the urban landscape, making them difficult to discover.
Our solution was to collect comprehensive data on galleries and exhibitions, presenting them on an interactive map.
However, to elevate the user experience beyond a simple map interface, we incorporated an AI-powered recommendation system.

Key Features:

- Personalized Preference Test: Users begin with a quick, engaging test where they select preferences for colors, textures, and sounds.

- AI-Driven Recommendations: Based on the user's choices, our AI system recommends specific galleries from our extensive database.

- Custom Route Creation: The app generates personalized routes for users to explore their recommended galleries efficiently.

- Proximity-Based Notifications: As users approach a gallery, the app sends push notifications with intriguing information. These might include viewing tips, historical facts about the gallery, or other engaging content to enhance the visitor's experience before they even step inside.

## Backend Architecture
We built our entire infrastructure on AWS, initially planning to use ECS Fargate to run our API server. The architecture is structured as follows:
![er graph](er_graph.png)
_architecture graph_
