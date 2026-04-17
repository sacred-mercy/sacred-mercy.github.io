---
icon: fas fa-laptop-code
order: 3
---

{% for project in site.data.projects %}
### {{ project.name }}

{{ project.description }}

{% for tag in project.tags %}`{{ tag }}`{% unless forloop.last %} · {% endunless %}{% endfor %} &nbsp;·&nbsp; [Live Demo]({{ project.url }}){:target="_blank" rel="noopener noreferrer"}

---
{% endfor %}
