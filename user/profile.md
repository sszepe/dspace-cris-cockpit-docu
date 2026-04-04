---
layout: page
title: Profile
permalink: /user/profile/
parent: User Guide
---

# My Profile

Navigate to **Profile** (`#/profile`) to view your account information and group memberships. This data comes directly from DSpace and is **read-only** in the Cockpit — to change your name, email, or password, use the DSpace administration interface.

---

## Profile Page Contents

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">My Profile</span>
  </div>
  <div style="padding:16px;">
    <div style="display:flex;align-items:center;gap:10px;margin-bottom:6px;">
      <div style="font-size:20px;font-weight:700;color:#e2e8f0;">My Profile</div>
      <span style="background:rgba(99,102,241,0.15);color:#818cf8;border:1px solid rgba(99,102,241,0.3);border-radius:12px;padding:2px 10px;font-size:11px;font-weight:700;">Administrator</span>
    </div>
    <div style="font-size:13px;color:#7a8599;margin-bottom:20px;">Account data and group memberships from DSpace CRIS.</div>
    <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;margin-bottom:12px;">
      <div style="font-size:13px;font-weight:600;color:#e2e8f0;margin-bottom:12px;padding-bottom:8px;border-bottom:1px solid #2a3145;">Account</div>
      <div style="display:grid;grid-template-columns:130px 1fr;gap:8px 12px;font-size:13px;">
        <div style="color:#4a5568;">Display name</div><div style="color:#e2e8f0;">Dr. Jane Smith</div>
        <div style="color:#4a5568;">Email</div><div style="color:#e2e8f0;">j.smith@university.edu</div>
        <div style="color:#4a5568;">NetID</div><div style="color:#e2e8f0;">jsmith</div>
        <div style="color:#4a5568;">UUID</div><div style="color:#7a8599;font-family:monospace;font-size:12px;">a1b2c3d4-e5f6-...</div>
        <div style="color:#4a5568;">Last active</div><div style="color:#7a8599;">2 minutes ago</div>
        <div style="color:#4a5568;">Can log in</div><div style="color:#3ecf8e;">Yes</div>
      </div>
    </div>
    <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;">
      <div style="font-size:13px;font-weight:600;color:#e2e8f0;margin-bottom:12px;padding-bottom:8px;border-bottom:1px solid #2a3145;">Groups <span style="background:#2a3145;color:#7a8599;border-radius:12px;padding:1px 8px;font-size:11px;margin-left:6px;">3</span></div>
      <div style="display:flex;flex-direction:column;gap:8px;font-size:13px;">
        <div style="display:flex;align-items:center;justify-content:space-between;">
          <span style="color:#e2e8f0;">Administrator</span>
          <div style="display:flex;gap:5px;">
            <span style="background:rgba(245,158,11,0.1);color:#f59e0b;border-radius:999px;padding:2px 8px;font-size:11px;font-weight:600;">Permanent</span>
            <span style="background:rgba(99,102,241,0.15);color:#818cf8;border-radius:999px;padding:2px 8px;font-size:11px;font-weight:600;">Administrator</span>
          </div>
        </div>
        <div style="display:flex;align-items:center;justify-content:space-between;">
          <span style="color:#e2e8f0;">Anonymous</span>
          <span style="background:rgba(245,158,11,0.1);color:#f59e0b;border-radius:999px;padding:2px 8px;font-size:11px;font-weight:600;">Permanent</span>
        </div>
      </div>
    </div>
  </div>
</div>

### Account Section

| Field | Description |
|---|---|
| Display name | Full name from your DSpace EPerson record |
| Email | Your account email address |
| NetID | Institutional NetID (if set in DSpace) |
| UUID | Your unique DSpace identifier |
| Last active | Timestamp of your last recorded activity |
| Can log in | Whether your account is active |
| Self registered | Whether you self-registered (vs. provisioned by an admin) |

### Community Admin Section

Only shown if you administer at least one community. Lists each community name and UUID that you have admin rights over. This section is determined by membership in `COMMUNITY_<uuid>_ADMIN` groups in DSpace.

### Groups Section

All DSpace groups you belong to, with role badges:

| Badge | Colour | Meaning |
|---|---|---|
| `Permanent` | Amber | A DSpace system group that cannot be deleted |
| `Administrator` | Indigo | You are in the global DSpace Administrator group |
| Entity type label | Indigo | The group is linked to a specific entity type |

---

## Role Badges

A coloured badge appears next to your name at the top of the profile page:

| Badge | Colour | What it means |
|---|---|---|
| `Administrator` | Indigo | You are in the DSpace `Administrator` group — full admin access to all features |
| `Community Admin` | Teal | You administer at least one community — you see creation and role management buttons for those communities |
| _(no badge)_ | — | Standard user — workspace, search, and browse access only |

---

## Changing Account Details

The profile page in the Cockpit is **read-only**. To change your name, email address, or password:

- Log in to the **DSpace administration UI** directly (ask your repository manager for the URL)
- Or contact your institution's repository manager and ask them to update your EPerson record

---

## Terms of Use {#terms}

If your institution has enabled the End User Agreement, a **Terms of Use** modal appears on your first login.

<div class="ui-mock">
  <div style="padding:24px 28px 0;border-bottom:1px solid #2a3145;">
    <div style="font-size:17px;font-weight:700;color:#e2e8f0;margin-bottom:4px;">Terms of Use</div>
    <div style="font-size:13px;color:#7a8599;padding-bottom:16px;">Please read and accept the terms of use to continue.</div>
  </div>
  <div style="padding:16px 28px;max-height:80px;overflow:hidden;font-size:13px;color:#7a8599;border-bottom:1px solid #2a3145;">
    <strong style="color:#e2e8f0;">1. Scope of use</strong><br>
    This repository is provided for the purposes of academic research and scholarship. By using this service…
    <div style="background:linear-gradient(transparent,#161b27);position:relative;height:30px;margin-top:-30px;"></div>
  </div>
  <div style="padding:16px 28px 20px;">
    <label style="display:flex;align-items:flex-start;gap:10px;cursor:pointer;font-size:13px;color:#7a8599;margin-bottom:14px;">
      <input type="checkbox" style="margin-top:3px;flex-shrink:0;"> I have read and agree to the terms of use.
    </label>
    <div style="display:flex;justify-content:flex-end;">
      <span style="background:#d1d5db;color:#9ca3af;border:none;border-radius:6px;padding:9px 24px;font-size:14px;font-weight:600;">Accept &amp; Continue</span>
    </div>
  </div>
</div>

<ol class="steps">
<li><div><strong>Read the Terms of Use</strong><br>Scroll through the full text. The content is set by your institution and stored in DSpace's site metadata.</div></li>
<li><div><strong>Tick the checkbox</strong><br>"I have read and agree to the terms of use." The Accept button only becomes active after ticking this box.</div></li>
<li><div><strong>Click Accept &amp; Continue</strong><br>Your acceptance is recorded on your DSpace account. The modal will not appear again on future logins.</div></li>
</ol>

<div class="callout callout-warn">
<span class="callout-title">You cannot skip or dismiss the Terms of Use modal</span>
If shown, you must accept the terms before you can access any part of the Cockpit. There is no dismiss or cancel option. If you have concerns about the terms, contact your institution's repository manager before accepting.
</div>

<div class="page-nav">
  <a href="{{ '/user/communities/' | relative_url }}">← Communities</a>
  <a href="{{ '/user/faq/' | relative_url }}">FAQ & Glossary →</a>
</div>
