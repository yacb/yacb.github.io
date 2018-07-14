<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }} - ({{ post.author }})</a>
    </li>
  {% endfor %}
</ul>
