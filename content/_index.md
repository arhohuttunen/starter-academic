---
# Leave the homepage title empty to use the site title
title:
date: 2023-09-23
type: landing

sections:
  - block: hero
    content:
      title: Learn to Build Software That Is Easy to Maintain
      image:
        filename: home.png
      cta:
        label: Visit My Channel
        url: https://youtube.com/@codewitharho
        icon_pack: fab
        icon: youtube
      cta_alt:
        label: Check out the wall of love
        url: "#section-testimonials"
      text: |-
        > Complex topics. Understandable guides.
      
        Easy to digest software development tutorials, and tips on how to become a better software developer. Save time, frustration and do a better job.
      
        Specialized in Java/Kotlin and Spring Boot.
        
        <p></p>
      cta_note:
        label: Check out my YouTube channel if you prefer video tutorials.
  - block: markdown
    content:
      text: |-
        <div style="text-align: center; margin: auto;" class="col-8">
        <h2>Clean Code Corner</h2>
        
        Short, simple, and actionable tips on developing quality software.
        
        <script async data-uid="c5699c15a9" src="https://arhohuttunen.ck.page/c5699c15a9/index.js"></script>
        
        <small>Join for free. No spam. Unsubscribe at any time.</small>
        </div>
    design:
      columns: '1'
  - block: testimonials
    content:
      title: Wall of Love
      items:
        - name: Jonathan Aplacador
          text: Please create more videos in Spring I learn so much with a short video of yours compared to hours of the same topic.
        - name: You Kaba
          text: Man this is awesome video. Please continue on that. Explanations are very simple, clear and helpful. Keep it up and thanks for sharing.
        - name: Евгений Кольяков
          text: Best series about testing in Spring. Looking forward for next videos.
        - name: Luis Javier Palacio Mesa
          text: Great videos, you should write a book, it would be awesome
        - name: Manuel Rivas
          text: Thank you man! I saw this and previous video, and I learned a lot. I have been programming with Spring since 1 year, but I have not payed attention to testing, and now, I clearly see that it's so important and useful.
        - name: Himanshu Ranjan
          text: This is one of the best youtube channel for quality of code content. Congratulations Arho for your efforts.
        - name: Linh Vũ
          text: Awesome, your explanation is very helpful, especially the last part, I've been struggling with a lot of annotations is used in test so far, but I get it better now, thank you!
        - name: Shasmita Gupta
          text: Well explained. Haven't seen anything better on this topic anywhere. Kudos!
        - name: Gerard Dróżdż
          text: Great video! A lot of useful parts and examples, especially when it comes to testing, bad request handling etc. You don't see it often enough in most Spring tutorials. Came here to see how to use Spring with Kotlin, but I see you have a lot of videos on testing with Spring, I'll definitely come back later to see them.
        - name: Farhad Yousefi
          text: I almost read all of your articles about tests. they are amazing, many thanks for sharing them with us.
  - block: collection
    id: posts
    content:
      title: Recent Articles
      subtitle: ''
      text: ''
      # Choose how many pages you would like to display (0 = all pages)
      count: 5
      # Filter on criteria
      filters:
        folders:
          - post
        author: ""
        category: ""
        tag: ""
        exclude_featured: false
        exclude_future: false
        exclude_past: false
        publication_type: ""
      # Choose how many pages you would like to offset by
      offset: 0
      # Page order: descending (desc) or ascending (asc) date.
      order: desc
    design:
      # Choose a layout view
      view: compact
      columns: '2'
  - block: collection
    id: featured
    content:
      title: Featured Articles
      subtitle: Trending Now
      filters:
        folders:
          - post
        featured_only: true
    design:
      columns: '2'
      view: compact
---
