---
layout: page
permalink: /repositories/
title: Repositories
description: Selected GitHub repositories with live repository stats.
nav: true
nav_order: 2
---

{% if site.data.repositories.github_users %}

## GitHub Users

<div id="live-github-users" class="live-repositories" aria-live="polite">
  <p>Loading GitHub user stats...</p>
</div>

{% endif %}

{% if site.data.repositories.github_repos %}

## Selected Repositories

<div id="live-repositories" class="live-repositories" aria-live="polite">
  <p>Loading repository stats...</p>
</div>

{% endif %}

<style>
  .live-repositories {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 1.25rem;
    margin-bottom: 2rem;
  }

  .live-repo-card {
    border: 1px solid var(--global-divider-color);
    border-radius: 1rem;
    box-shadow: 0 0.35rem 1.25rem rgba(0, 0, 0, 0.04);
    background: var(--global-card-bg-color);
    padding: 1.15rem;
    transition:
      border-color 0.2s ease,
      box-shadow 0.2s ease,
      transform 0.2s ease;
  }

  .live-repo-card:hover {
    border-color: var(--global-theme-color);
    box-shadow: 0 0.55rem 1.75rem rgba(0, 0, 0, 0.08);
    transform: translateY(-2px);
  }

  .live-user-card {
    border-left: 4px solid var(--global-theme-color);
  }

  .live-user-header {
    align-items: center;
    display: flex;
    gap: 0.85rem;
    margin-bottom: 0.75rem;
  }

  .live-user-avatar {
    border: 2px solid var(--global-divider-color);
    border-radius: 999px;
    height: 3rem;
    width: 3rem;
  }

  .live-repo-card h3 {
    font-size: 1.1rem;
    margin-bottom: 0.5rem;
  }

  .live-repo-description {
    min-height: 3rem;
    color: var(--global-text-color);
  }

  .live-repo-stats {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 0.75rem;
    font-size: 0.95rem;
  }

  .live-repo-stats span {
    align-items: center;
    background: color-mix(in srgb, var(--global-theme-color) 10%, transparent);
    border: 1px solid color-mix(in srgb, var(--global-theme-color) 22%, transparent);
    border-radius: 999px;
    display: inline-flex;
    gap: 0.35rem;
    line-height: 1;
    padding: 0.4rem 0.6rem;
  }

  .live-repo-stats i {
    color: var(--global-theme-color);
  }

  .live-repo-meta {
    align-items: center;
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
    margin-top: 0.75rem;
    font-size: 0.85rem;
    color: var(--global-text-color-light);
  }

  .live-language-dot {
    background: var(--global-theme-color);
    border-radius: 999px;
    display: inline-block;
    height: 0.65rem;
    width: 0.65rem;
  }
</style>

<script>
  (() => {
    const githubUsers = {{ site.data.repositories.github_users | jsonify }};
    const repositories = {{ site.data.repositories.github_repos | jsonify }};
    const usersContainer = document.getElementById("live-github-users");
    const reposContainer = document.getElementById("live-repositories");
    const formatter = new Intl.NumberFormat();

    const escapeHtml = (value) =>
      String(value ?? "")
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#039;");

    const formatDate = (value) =>
      new Intl.DateTimeFormat("en", {
        year: "numeric",
        month: "short",
        day: "numeric",
      }).format(new Date(value));

    const fetchJson = (url) =>
      fetch(url).then((response) => {
        if (!response.ok) throw new Error(`GitHub API failed for ${url}`);
        return response.json();
      });

    const fetchUserRepos = async (username) => {
      const repos = [];
      let page = 1;

      while (true) {
        const batch = await fetchJson(
          `https://api.github.com/users/${username}/repos?type=owner&per_page=100&page=${page}`
        );
        repos.push(...batch);
        if (batch.length < 100) break;
        page += 1;
      }

      return repos;
    };

    const renderUser = async (username) => {
      const [user, ownedRepos, selectedRepos] = await Promise.all([
        fetchJson(`https://api.github.com/users/${username}`),
        fetchUserRepos(username),
        Promise.all(repositories.map((repo) => fetchJson(`https://api.github.com/repos/${repo}`).catch(() => null))),
      ]);

      const reposForTotals = [...ownedRepos, ...selectedRepos.filter(Boolean)].filter(
        (repo, index, repos) => repos.findIndex((candidate) => candidate.full_name.toLowerCase() === repo.full_name.toLowerCase()) === index
      );

      const totals = reposForTotals.reduce(
        (sum, repo) => ({
          stars: sum.stars + repo.stargazers_count,
          forks: sum.forks + repo.forks_count,
          issues: sum.issues + repo.open_issues_count,
        }),
        { stars: 0, forks: 0, issues: 0 }
      );

      return `
        <article class="live-repo-card live-user-card">
          <div class="live-user-header">
            <img class="live-user-avatar" src="${user.avatar_url}" alt="${escapeHtml(user.login)} avatar">
            <div>
              <h3><a href="${user.html_url}" target="_blank" rel="noopener noreferrer">${escapeHtml(user.login)}</a></h3>
              <div class="live-repo-meta">GitHub profile</div>
            </div>
          </div>
          <p class="live-repo-description">${escapeHtml(user.bio || "GitHub profile summary.")}</p>
          <div class="live-repo-stats">
            <span title="Total stars" aria-label="Total stars"><i class="fa-solid fa-star" aria-hidden="true"></i> ${formatter.format(totals.stars)}</span>
            <span title="Total forks" aria-label="Total forks"><i class="fa-solid fa-code-fork" aria-hidden="true"></i> ${formatter.format(totals.forks)}</span>
            <span title="Followers" aria-label="Followers"><i class="fa-solid fa-user-group" aria-hidden="true"></i> ${formatter.format(user.followers)}</span>
          </div>
          <div class="live-repo-meta">
            Total stats across owned public repositories and selected main-contributor repositories.
          </div>
        </article>
      `;
    };

    const renderRepo = (repo) => `
      <article class="live-repo-card">
        <h3><a href="${repo.html_url}" target="_blank" rel="noopener noreferrer">${escapeHtml(repo.full_name)}</a></h3>
        <p class="live-repo-description">${escapeHtml(repo.description || "No description provided.")}</p>
        <div class="live-repo-stats">
          <span title="Stars" aria-label="Stars"><i class="fa-solid fa-star" aria-hidden="true"></i> ${formatter.format(repo.stargazers_count)}</span>
          <span title="Forks" aria-label="Forks"><i class="fa-solid fa-code-fork" aria-hidden="true"></i> ${formatter.format(repo.forks_count)}</span>
          <span title="Watchers" aria-label="Watchers"><i class="fa-solid fa-eye" aria-hidden="true"></i> ${formatter.format(repo.subscribers_count ?? repo.watchers_count)}</span>
          <span title="Open issues" aria-label="Open issues"><i class="fa-solid fa-circle-dot" aria-hidden="true"></i> ${formatter.format(repo.open_issues_count)}</span>
        </div>
        <div class="live-repo-meta">
          ${repo.language ? `<span class="live-language-dot" aria-hidden="true"></span><span>${escapeHtml(repo.language)}</span>` : ""}
          <span>Updated ${formatDate(repo.updated_at)}</span>
        </div>
      </article>
    `;

    if (usersContainer && githubUsers?.length) {
      Promise.all(
        githubUsers.map((username) =>
          renderUser(username).catch(
            () => `
              <article class="live-repo-card live-user-card">
                <h3><a href="https://github.com/${username}" target="_blank" rel="noopener noreferrer">${escapeHtml(username)}</a></h3>
                <p class="live-repo-description">Could not load live GitHub user stats. Open the profile on GitHub for the latest numbers.</p>
              </article>
            `
          )
        )
      ).then((cards) => {
        usersContainer.innerHTML = cards.join("");
      });
    }

    if (reposContainer && repositories?.length) {
      Promise.all(
        repositories.map((repo) =>
          fetchJson(`https://api.github.com/repos/${repo}`)
            .then(renderRepo)
            .catch(
              () => `
                <article class="live-repo-card">
                  <h3><a href="https://github.com/${repo}" target="_blank" rel="noopener noreferrer">${escapeHtml(repo)}</a></h3>
                  <p class="live-repo-description">Could not load live GitHub stats. Open the repository on GitHub for the latest numbers.</p>
                </article>
              `
            )
        )
      ).then((cards) => {
        reposContainer.innerHTML = cards.join("");
      });
    }
  })();
</script>
