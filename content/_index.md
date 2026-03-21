---
title: 'Home'
type: landing
sections:
  - block: resume-biography
    content:
      # Author slug (data/authors/<slug>.yaml)
      username: admin
    design:
      spacing:
        padding: [0, 0, 0, 0]
      biography:
        style: 'text-align: center; font-size: 0.8em;'
      # Avatar customization
      avatar:
        size: medium  # Options: small (150px), medium (200px, default), large (320px), xl (400px), xxl (500px)
        shape: circle # Options: circle (default), square, rounded
  - block: collection
    content:
      filters:
        folders:
          - blog
    design:
      view: compact
      spacing:
        padding: ['3rem', 0, '6rem', 0]
---
