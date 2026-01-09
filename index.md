---
layout: default
title: Home
---

<section id="home" class="hero">
  <div class="hero-eyebrow">$WHOAMI</div>
  <h1><span class="typewriter" data-text="Wiljohn."></span><span class="cursor">|</span></h1>
  <div class="hero-subtitle">I study the art of breaking in to master the science of keeping them out.</div>
  <div class="hero-bio">
    A SOC Analyst. I thrive on blue team defense and creative problem-solving. Always learning, always curious.
  </div>
</section>

<section id="experience" class="experience">
  <h2>Experience</h2>
  <div class="experience-tabs">
    <div class="job-list">
      <div class="job-item  active" data-bullets="Triaged and analyzed over 50 daily security alerts to identify and mitigate potential threats.|Coordinated effectively with internal teams to contain, escalate, and resolve security incidents promptly.|Ensured consistent policy compliance and assisted with the fine-tuning of alert rules based on real-world threat patterns and intelligence.">
        <h3>SOC 1 Analyst</h3>
        <div class="company">Philcox Inc. | Quezon City, NCR, Philippines</div>
        <div class="period">February 2025 - Present</div>
      </div>
      <div class="job-item" data-bullets="Provided comprehensive remote hardware and software support, including system configuration and troubleshooting for a global user base.|Authored and maintained Standard Operating Procedures (SOPs) to improve internal knowledge sharing and enhance the consistency of IT service delivery.|Escalated unresolved incidents to higher-tier engineers while maintaining strict Service Level Agreements (SLAs) to ensure timely resolution.">
        <h3>Global Remote Support</h3>
        <div class="company">Collabera Digital | Quezon City, NCR, Philippines</div>
        <div class="period">September 2022 - February 2025</div>
      </div>
    </div>
    <div class="job-details">
      <h3>SOC 1 Analyst</h3>
      <div class="role">Philcox Inc. | Quezon City, NCR, Philippines</div>
      <ul>
        <li>Triaged and analyzed over 50 daily security alerts  to identify and mitigate potential threats.</li>
        <li>Coordinated effectively with internal teams to contain, escalate, and resolve security incidents promptly.</li>
        <li>Ensured consistent policy compliance and assisted with the fine-tuning of alert rules based on real-world threat patterns and intelligence.</li>
      </ul>
    </div>
  </div>
</section>

<section id="certifications" class="certifications">
  <h2>Certifications</h2>
  <div class="certifications-grid">
    <div class="certification-card">
      <div class="cert-icon">🛡️</div>
      <h3>ISC2 Certified in Cybersecurity (CC)</h3>
      <div class="cert-issuer">ISC²</div>
      <div class="cert-date">May 1, 2025 - Apr 30, 2028</div>
      <div class="cert-status valid">Valid</div>
    </div>
    <div class="certification-card">
      <div class="cert-icon">🔵</div>
      <h3>Blue Team Level 1</h3>
      <div class="cert-issuer">Security Blue Team</div>
      <div class="cert-date">April 04, 2025 - No Expiration</div>
      <div class="cert-status valid">Valid</div>
    </div>
    <div class="certification-card">
      <div class="cert-icon">🔒</div>
      <h3>CompTIA CySa+</h3>
      <div class="cert-issuer">CompTIA</div>
      <div class="cert-date">May 27, 2025 - May 27, 2028</div>
      <div class="cert-status valid">Valid</div>
    </div>
    <div class="certification-card">
      <div class="cert-icon">🎯</div>
      <h3>Junior Penetration Tester</h3>
      <div class="cert-issuer">INE.com</div>
      <div class="cert-date">September 09, 2025 - September 08, 2028</div>
      <div class="cert-status valid">Valid</div>
    </div>
  </div>
</section>

<section id="writeups" class="writeups">
  <h2>Write-ups</h2>
  <div class="writeups-grid">
    {% assign random_posts = site.posts | sample: 3 %}
      {% for post in random_posts %}
      <a href="/gooduck/{{ post.url }}" class="writeup-card" style="text-decoration:none; color:inherit;">
        <h3>{{ post.title }}</h3>
        <p>{{ post.excerpt }}</p>
        <div class="writeup-tags">
          {% for category in post.categories %}
          <span class="writeup-tag">{{ category }}</span>
          {% endfor %}
        </div>
        <span class="read-more">Read More →</span>
      </a>
      {% endfor %}
  </div>
  <div class="writeups-all-btn-container" style="text-align:center; margin-top:2em;">
    <a href="/gooduck/write-ups.html" class="writeups-all-btn" style="display:inline-block; padding:0.75em 2em; background:#222; color:#fff; border-radius:4px; text-decoration:none; font-weight:bold;">View All Write-ups</a>
  </div>
  </section>