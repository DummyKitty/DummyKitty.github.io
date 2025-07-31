---
title: FRIENDS
icon: fa-fw fas fa-user-group
order: 5
---

<style>
.friends-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  margin: 20px 0;
}

.friend-card {
  background: var(--card-bg);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  display: flex;
  align-items: center;
}

.friend-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 16px rgba(0,0,0,0.15);
}

/* 头像样式 */
.friend-avatar {
  margin-right: 15px;
  flex-shrink: 0; /* 防止头像容器被压缩 */
}

.friend-avatar img {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover; /* 确保图片按比例裁剪填充 */
  display: block;
}

/* 友链信息样式 */
.friend-info {
  flex: 1;
  min-width: 0; /* 允许文字内容压缩 */
}

.friend-name {
  margin: 0 0 8px 0;
  font-size: 1.1em;
}

.friend-name a {
  text-decoration: none;
  color: var(--text-color);
}

.friend-name a:hover {
  color: var(--link-color);
}

.friend-desc {
  margin: 0;
  font-size: 0.9em;
  color: var(--text-muted);
  line-height: 1.4;
}
</style>

<div class="friends-container">
{% for link in site.data.links %}
  <div class="friend-card">
    <div class="friend-avatar">
      <img src="{{ link.avatar }}" alt="{{ link.name }}" onerror="this.src='default-avator.jpg'">
    </div>
    <div class="friend-info">
      <h3 class="friend-name">
        <a href="{{ link.url }}" target="_blank">{{ link.name }}</a>
      </h3>
      <p class="friend-desc">{{ link.description }}</p>
    </div>
  </div>
{% endfor %}
</div>
