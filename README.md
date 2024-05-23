Hi and welcome!

Here you will find various articles related to topics I am
interested in such as property based testing, fuzzing and
other software development techniques.

You can find the slides for my presentation about 
property based testing that I held at EDC 2023 [here](https://github.com/EJahren/property-based-testing-slides).

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
