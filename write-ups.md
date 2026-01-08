---
layout: write-up
title: Write-Ups
---

# Write-Ups

A collection of recent write-ups and CTF walkthroughs.

<input type="text" id="writeup-search" class="writeup-search" placeholder="Search write-ups..." autocomplete="off" style="width:100%;max-width:400px;margin:2rem 0 2rem 0;display:block;padding:0.75rem 1rem;border-radius:8px;border:1px solid var(--bg-secondary);background:var(--bg-secondary);color:var(--text-primary);font-size:1rem;">

<div class="writeups-grid">
{% for post in site.posts %}
  <a href="{{ post.url }}" class="writeup-card-link" data-title="{{ post.title | escape }}" data-excerpt="{{ post.excerpt | escape }}" data-tags="{% for tag in post.tags %}{{ tag | escape }} {% endfor %}">
    <article class="writeup-card">
      {% if post.image %}
        <img src="{{ post.image }}" alt="{{ post.title }}" class="writeup-thumb">
      {% endif %}
      <div class="writeup-body">
        <h3>{{ post.title }}</h3>
        <p class="writeup-meta"><small>{{ post.date | date: "%B %d, %Y" }} </small></p>
        <p class="writeup-excerpt">{{ post.excerpt }}</p>
        <div class="writeup-tags">
          {% for tag in post.tags %}
          <span class="writeup-tag">{{ tag }}</span>
          {% endfor %}
        </div>
        <span class="read-more">Read More →</span>
      </div>
    </article>
  </a>
{% endfor %}
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
  const searchInput = document.getElementById('writeup-search');
  const cards = Array.from(document.querySelectorAll('.writeup-card-link'));
  searchInput.addEventListener('input', function() {
    const q = this.value.trim().toLowerCase();
    cards.forEach(card => {
      const title = (card.getAttribute('data-title') || '').toLowerCase();
      const excerpt = (card.getAttribute('data-excerpt') || '').toLowerCase();
      const tags = (card.getAttribute('data-tags') || '').toLowerCase();
      if (title.includes(q) || excerpt.includes(q) || tags.includes(q)) {
        card.style.display = '';
      } else {
        card.style.display = 'none';
      }
    });
  });
});
</script>