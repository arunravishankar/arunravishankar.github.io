---
layout: default
---

### Peer-reviewed Writing

1. M.Brozen, T.Black, R.Liggett: Comparing Measures and Variables in Multimodal Street Performance Calculations ( [paper](http://trrjournalonline.trb.org/doi/10.3141/2420-01) )
2. T.Black, J.Swartz, T.Fremaux - Vision Zero and Beyond: A Simple Yet Powerful Data Strategy for Evaluating Potential Engineering Solutions ( [paper](documents/TRB2017_VisionZeroBeyond_Paper.pdf) / [poster](documents/TRB2017_VisionZeroBeyond_Poster.pdf) )
3. T.Black - Redefining "Transportation Impact": A Comparison of Emerging Methodologies ( [paper](documents/TRB2015_SB743_Paper.pdf) / [slides](documents/TRB2015_SB743_Slides.pdf) )
4. M.Brozen, T.Black, H.Huff, R.Liggett - Multimodal Street Performance Measure Sensitivity Case Study: How to Get an A ( [poster](documents/TRB2015_MMLOS_Poster.pdf) )

<div id="articles">
{% for category in site.categories %}
  <h3>{{ category }}</h3>
  <ul>
    {% for posts in category %}
      {% for post in posts %}
        {% if post.url %}
          <li><a href="{{ post.url }}">{{ post.title }}</a></li>
        {% endif %}
      {% endfor %}
    {% endfor %}
    </ul>
{% endfor %}
</div>

---
