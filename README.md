# Liferay Forums

This Liferay Workspace project is a Fragments and Liferay Objects based replacement for the legacy Message Boards / Questions widgets which are deprecated.

The project is delivered as **two site initializers** that build two sites with a clear separation of concerns:

- **Forum Example Site** (`forums-site-initializer`) — the public, end-user experience: the forums home, the discussions list, thread display pages and the New Discussion page. It runs on the Classic theme.
- **Forums Admin** (`forums-admin-site-initializer`) — the administration experience (category management and moderation), presented like a standalone product using the same shell and styling as **Liferay CMS** (a left product menu, a top product bar with the application switcher, and the CMS theme).

Both sites share the same data because the Forum Objects are **company-scoped**: a single dataset is administered from Forums Admin and surfaced on the Forum Example Site. See [Two-Site Architecture](#two-site-architecture) below.

---

## Screenshots

<table>
  <tr>
    <td align="center"><strong>Forums Home</strong></td>
    <td align="center"><strong>Message List</strong></td>
    <td align="center"><strong>Message Detail</strong></td>
  </tr>
  <tr>
    <td><a href="screenshots/screenshot-forums.png"><img src="screenshots/screenshot-forums.png" width="260" alt="Forums home page"></a></td>
    <td><a href="screenshots/screenshot-message-list.png"><img src="screenshots/screenshot-message-list.png" width="260" alt="Message list view"></a></td>
    <td><a href="screenshots/screenshot-message-detail.png"><img src="screenshots/screenshot-message-detail.png" width="260" alt="Message detail view"></a></td>
  </tr>
</table>

---

## Setup

The project produces **two** site-initializer client extensions under `client-extensions/`:

| Client extension | Site ERC | Site name | Friendly URL |
| :--- | :--- | :--- | :--- |
| `forums-site-initializer` | `FORUMS` | `Forum Example Site` | `/forum-example-site` |
| `forums-admin-site-initializer` | `FORUMSADMIN` | `Forums Admin` | `/forums-admin` |

The site ERC, name and other values can be changed in each extension's `client-extension.yaml`:

```yaml
    siteExternalReferenceCode: FORUMS
    siteName: Forum Example Site
```

Build them with the standard Liferay Workspace wrapper commands (e.g., `./gw build`) and deploy each by copying the resulting artifact to `$LIFERAY_HOME/deploy`.

> **Deploy order matters.** The Forums Admin site composes the shared, company-scoped fragments (the admin fragments and the CMS-style chrome) that are imported by `forums-site-initializer`. Deploy `forums-site-initializer` **first** so those fragments exist when the admin site initializes.

> **Note on renaming.** Renaming a site changes its auto-derived group friendly URL (renaming to "Forum Example Site" yields `/forum-example-site`). Account for this if you hardcode links; the fragments themselves derive the site path at runtime.

| File | Description |
| :--- | :--- |
| [setup-forum-permissions.groovy](scripts/_01_setup/setup-forum-permissions.groovy) | Groovy script that grants the required Object permissions to the Guest and Site Member roles. Run via **Control Panel → Server Administration → Script**. This step is necessary because Liferay Objects has no equivalent to the `<resource-action-mapping>` XML descriptor used by Service Builder to define default permissions — Object permissions must be configured explicitly after import. The Headless REST API does not have endpoints that support this yet. It also sets up a Service Access Policy so that non-authenticated users can invoke the REST APIs in order to see forum messages and replies. |

---

## Required Feature Flags

The following **Release** feature flags must be enabled before deploying.

| Ticket | Description |
| :--- | :--- |
| LPD-17564 | CMS |
| LPD-34594 | Root Object Definitions |
| LPD-11235 | Enhanced Rich Text Editor |

---

## Two-Site Architecture

Administration is separated from the public experience into two sites, following the pattern Liferay CMS uses to present a site as a standalone admin product.

```
Forum Example Site (/forum-example-site)        Forums Admin (/forums-admin)
Classic theme, end users                         CMS theme, administrators
├── Forums            (home)                     ├── Categories   (manage categories)
├── Forums Messages   (discussions list)         └── Moderation   (review flagged content)
└── New Discussion    (hidden)
    + ForumMessage / ForumReply display pages
```

### Company-scoped data model

All nine Forum Object definitions use **company** scope (not site). With company scope the generated REST endpoints have **no** `/scopes/{groupId}` segment — entries are read and written at `/o/c/<object>` (e.g. `/o/c/forumcategories`). This is what lets a single dataset be administered on Forums Admin and rendered on the Forum Example Site.

Consequences worth knowing:

- Fragment calls (both client-side `fetch` and server-side FreeMarker `restClient.get`) use `/o/c/<object>`, never `/o/c/<object>/scopes/...`.
- Object scope is **immutable on a published definition**. Changing it requires deleting the object relationships and definitions (and their data) and re-initializing.
- Because the data is company-wide, it **survives deleting and recreating either site**, so iterating on a site no longer wipes forum content.
- Links to an object's display page derive the site path from the **current page URL** (the moderation fragment, which runs on Forums Admin where the display page does not exist, instead targets the example site via its **Messages Site Friendly URL** configuration field).

### CMS-style admin shell

`forums-admin-site-initializer` reproduces the CMS product look with declarative artifacts only:

- **CMS theme** (`layout-set/public/metadata.json` → `"themeName": "CMS"`) and a hidden control menu (`site-configuration.json`).
- A **master page** (`forums-admin-master`) that composes four chrome fragments — `sidebar` (the CSS-grid shell), `page-bar` (top bar with the waffle logo, product title, application switcher and user bar), `sidebar-trigger` (hamburger) and `vertical-navigation` (the left menu, bound to the `FORUMSADMINNAV` site navigation menu) — around a content drop zone.
- Pages (`Categories`, `Moderation`) that embed the shared `forums-categories-admin` and `forums-moderation` fragments.

The chrome fragments live in the shared **company** fragment collection owned by `forums-site-initializer`, so they are referenced by key alone from the admin master page.

---

## Fragments

The fragments live in the shared company collection owned by `forums-site-initializer` (`site-initializer/fragments/company/forums/fragments/`).

### Forum fragments

| Fragment Name | Folder | Description |
| :--- | :--- | :--- |
| **Forums Categories Admin** | [forums-categories-admin](fragments/forums-categories-admin) | Administration interface for managing forum categories. Placed on the **Forums Admin** site. |
| **Forums Category Grid** | [forums-category-grid](fragments/forums-category-grid) | Displays the main forum categories in a grid layout. |
| **Forums Hero** | [forums-hero](fragments/forums-hero) | Top banner for the forums featuring statistics (like member count) and quick actions. |
| **Forums Moderation** | [forums-moderation](fragments/forums-moderation) | Tools for moderating forum content. Placed on the **Forums Admin** site. |
| **Forums Message Composer** | [forums-message-composer](fragments/forums-message-composer) | Composer for creating and editing forum messages and replies. Supports two `Form Mode`s: **modal** (replies/edits on the thread page) and **page** (the standalone New Discussion page). |
| **Forums Related Topics** | [forums-related-topics](fragments/forums-related-topics) | Displays a list of topics related to the currently viewed message. |
| **Forums Message Detail** | [forums-message-detail](fragments/forums-message-detail) | Detailed view of a single forum message, including its replies and engagement metrics. |
| **Forums Message List** | [forums-message-list](fragments/forums-message-list) | Lists forum messages, typically used for main category views or recent activity. |

### CMS-style chrome fragments

These power the Forums Admin product shell and are adapted from the Liferay CMS site initializer.

| Fragment Name | Folder | Description |
| :--- | :--- | :--- |
| **Page Bar** | [page-bar](fragments/page-bar) | Top product bar: waffle logo, product title, the Product Navigation Applications Menu portlet (application switcher) and the user personal bar. |
| **Sidebar** | [sidebar](fragments/sidebar) | The collapsible CSS-grid shell (left rail + content area) with the `topBar`, `sidebarBody` and `content` drop zones. |
| **Sidebar Trigger** | [sidebar-trigger](fragments/sidebar-trigger) | The hamburger button that collapses/expands the rail. |
| **Vertical Navigation** | [vertical-navigation](fragments/vertical-navigation) | Renders a site navigation menu as a Clay `menubar-primary` nav. Rendered entirely server-side (no React), so it needs no client bundling. |

---

## Page Layout and Fragment Placement

The forums application is assembled using standard pages and Display Page Templates. Each site initializer creates its own pages automatically — defined in `site-initializer/layouts/`. Layout directories are numeric-prefixed (`01_`, `02_`, …) so ordering is deterministic and the first page is the site's default landing page. The diagrams below preview how the fragments are arranged.

```
Forum Example Site  (/forum-example-site)        Forums Admin  (/forums-admin)
├── Forums            /forums          (home)    ├── Categories   /categories
├── Forums Messages   /forums-messages           └── Moderation   /moderation
└── New Discussion    /new-discussion  (hidden)
```

> The Forums Admin pages live on a dedicated administration site, so they are naturally separated from end users. You may still want to ***restrict the Forums Admin site membership/visibility*** to administrators, as site/page permissions cannot be set in a site initializer.

### Forum Example Site — Page: Forums
*Friendly URL: `/forums` — Main entry point and default landing page.*
```
┌─────────────────────────────────────────────────────────────────┐
│  forums-hero                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  drop-zone → Search Bar widget                            │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  forums-category-grid                                           │
└─────────────────────────────────────────────────────────────────┘
```

> The **Search Bar** widget is automatically present in the `forums-hero` drop-zone. ***Edit its configuration*** to specify the destination search page friendly URL (e.g. `/search`) and any other relevant search settings (scope, placeholder text, etc.).

### Forum Example Site — Page: Forums Messages
*Friendly URL: `/forums-messages` — The discussions list. Hidden from navigation.*
```
┌─────────────────────────────────────────────────────────────────┐
│  Fixed-width container (container-fluid-max-xl)                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  forums-message-list                                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Forum Example Site — Page: New Discussion
*Friendly URL: `/new-discussion` — Hidden. The composer in **page** mode.*
```
┌─────────────────────────────────────────────────────────────────┐
│  Fixed-width container (container-fluid-max-xl)                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  forums-message-composer  (Form Mode = page)              │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

> The "New Discussion" actions in the hero and the message list link here. Cancel goes back; a successful post redirects to the new thread's display page.

### Forums Admin — Page: Categories
*Friendly URL: `/categories` — Rendered inside the CMS-style admin shell.*
```
┌─────────────────────────────────────────────────────────────────┐
│  forums-categories-admin                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Forums Admin — Page: Moderation
*Friendly URL: `/moderation` — Rendered inside the CMS-style admin shell.*
```
┌─────────────────────────────────────────────────────────────────┐
│  forums-moderation                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Display Page Templates

The site initializer creates two Display Page Templates automatically — one mapped to `ForumMessage`, one mapped to `ForumReply`. Both use the identical fragment arrangement described below and are defined in `site-initializer/layout-page-templates/display-page-templates/`.

> **How Object Definition resolution works:** Liferay's raw Display Page Template export bakes in a volatile internal ID suffix (e.g. `com.liferay.object.model.ObjectDefinition#A0Z2`) as the `contentType.className`, which breaks on re-import whenever Object Definitions are recreated. The site initializer avoids this by using `BundleSiteInitializer`'s token replacement system. When Object Definitions are created during initialization, the method `_replaceObjectDefinitionValues` registers tokens like `OBJECT_DEFINITION_CLASS_NAME:ForumMessage` → `com.liferay.object.model.ObjectDefinition#xxxx`. The `display-page-template.json` uses these tokens:
> ```json
> {"contentType": {"className": "[$OBJECT_DEFINITION_CLASS_NAME:ForumMessage$]"}, "name": "Forum Message"}
> ```
> The `[$...$]` tokens are resolved at runtime before the layout importer processes the file, so the correct `className` (including its instance-specific `#suffix`) is always injected.

### Layout

Both Display Page Templates share the same structure:

```
┌─────────────────────────────────────────────────────────────────┐
│  Container                                                      │
│  ┌───────────────────────────────────────┬───────────────────┐  │
│  │  Column — 75%                         │  Column — 25%     │  │
│  │                                       │                   │  │
│  │  ┌─────────────────────────────────┐  │  ┌─────────────┐  │  │
│  │  │  forums-message-detail          │  │  │   forums-   │  │  │
│  │  └─────────────────────────────────┘  │  │  related-   │  │  │
│  │  ┌─────────────────────────────────┐  │  │   topics    │  │  │
│  │  │  forums-message-composer        │  │  └─────────────┘  │  │
│  │  └─────────────────────────────────┘  │                   │  │
│  └───────────────────────────────────────┴───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Inside the Container, a **2-column Grid** with a **75% / 25%** split:
- **Left column:** `forums-message-detail`, then `forums-message-composer`
- **Right column:** `forums-related-topics`

> **Important:** `forums-message-composer` must be present in every Display Page Template. Without it, users have no way to edit or reply to a message, because the composer modal is the sole entry point for both actions.

### ERC Field Mapping

The `forums-message-detail` and `forums-related-topics` fragments each contain hidden `div` elements that appear as mappable fields in the page editor but are not rendered to end-users at runtime. The site initializer's `page-definition.json` files pre-configure these mappings.

```html
<!-- Exposed as mappable fields in the Content Page Editor; hidden at runtime -->
<div id="forumsDetailERC"      data-lfr-editable-id="forumsDetailERC"      data-lfr-editable-type="text" style="display:none;">Mappable Message ERC</div>
<div id="forumsDetailReplyERC" data-lfr-editable-id="forumsDetailReplyERC" data-lfr-editable-type="text" style="display:none;">Mappable Reply ERC</div>
```

Each field is mapped to `ObjectEntry_externalReferenceCode` from the `DisplayPageItem` context, but only in the DPT that corresponds to its object type:

| Field | Forum Message DPT | Forum Reply DPT |
| :--- | :--- | :--- |
| **Mappable Message ERC** | Map to `ForumMessage → externalReferenceCode` | Leave unmapped |
| **Mappable Reply ERC** | Leave unmapped | Map to `ForumReply → externalReferenceCode` |

---

## Demo Data

The `scripts/_02_demo/` directory contains scripts for populating a development environment with forum content. The three scripts are numbered and should be run in order.

> **Company scope caveat.** These scripts predate the move to company-scoped Objects and address entries through `/o/c/<object>/scopes/{siteId}`. With company scope the correct path is `/o/c/<object>` (no `/scopes` segment, no `siteId`). Adjust the scripts accordingly, or seed a few entries directly against the company endpoints.

### Step 1 — Create demo data

```bash
python3 scripts/_02_demo/_01_create-demo-data.py <siteId> [BASE_URL] [--email EMAIL] [--password PASSWORD]
```

Creates the four default Forum Categories (by ERC if they do not already exist), a set of demo user accounts assigned the Site Member role with profile photos, Forum Messages with keywords distributed across those categories, and replies to each message authored by different users.

| Argument | Default | Description |
| :--- | :--- | :--- |
| `siteId` | *(required)* | Site (group) ID to scope all entries to |
| `BASE_URL` | `http://localhost:8080` | Liferay portal base URL |
| `--email` | `test@liferay.com` | Admin account email |
| `--password` | `test` | Admin account password |

### Step 2 — Backfill Forum Stats Users

Run [_02_backfill-forum-stats-users.groovy](scripts/_02_demo/_02_backfill-forum-stats-users.groovy) via **Control Panel → Server Administration → Script**.

Backfills `ForumStatsUser` records for every user who has posted a message or reply. Without these records, the `forums-hero` fragment displays 0 Members. If re-running, first clear existing records with `scripts/_03_util/delete-forum-stats-users.py` to avoid duplicates.

### Step 3 — Backfill create dates

Run [_03_backfill-create-dates.sql](scripts/_02_demo/_03_backfill-create-dates.sql) against the portal database.

Copies the `displayDate` values (set by Step 1) into the `createDate` and `modifiedDate` columns on the `objectentry` table for `ForumMessage` and `ForumReply` entries. This ensures that the entries appear with realistic chronological dates rather than all sharing the same import timestamp. After running, flush caches and rebuild indexes:

1. **Control Panel → Server Administration → Resources**
   - Clear content cached by this VM.
   - Clear content cached across the cluster.
   - Clear the database cache.
2. **Control Panel → Search → Index Actions**
   - Reindex all search indexes.

---

## Utilities

The `scripts/_03_util/` directory contains cleanup and teardown scripts.

| File | Description |
| :--- | :--- |
| [delete-demo-data.py](scripts/_03_util/delete-demo-data.py) | Deletes all Forum Stats User, Forum Reply, Forum Message, and Forum Category entries for a given site. Forum Votes are removed automatically via cascade. Demo user accounts are left in place. Usage: `python3 scripts/_03_util/delete-demo-data.py <siteId> [BASE_URL] [--email EMAIL] [--password PASSWORD]` |
| [delete-forum-stats-users.py](scripts/_03_util/delete-forum-stats-users.py) | Deletes all `ForumStatsUser` entries via the Objects REST API. Run this before re-executing Step 2 to ensure no duplicate records. Accepts an optional `--scope` argument (site `groupId` or friendly URL); if omitted the script auto-detects the correct scope. Usage: `python3 scripts/_03_util/delete-forum-stats-users.py [BASE_URL] [--scope SCOPE] [--email EMAIL] [--password PASSWORD]` |
| [delete-forum-object-definitions.py](scripts/_03_util/delete-forum-object-definitions.py) | Deletes all Object definitions whose name starts with `Forum` via the Object Admin REST API. Useful for fully resetting a dev environment. |

---

## Known Limitations

### View Count Not Incremented for Guest Users

The `forums-message-detail` fragment PATCHes the `viewCount` field on `ForumMessage` objects only for authenticated users. Guest views are silently skipped because the Liferay Object REST API returns `403 Forbidden` for unauthenticated PATCH requests.

**Option:** Grant the Guest role `update` permission on ForumMessage objects via the Object's permissions configuration. However, the preferred solution is a dedicated endpoint in a Spring Boot Client Extension that accepts a `messageId` and increments `viewCount` with its own service credentials — keeping the Object's permissions locked down.

### Ban Enforcement Is UI-Only

When a user is banned (a `ForumBan` Object entry exists for their user ID), the fragments detect this at page load by querying `GET /o/c/forumbans?filter=banUserId eq {userId}`. If a ban is found, the UI is locked down: the submit button is disabled, compose buttons are hidden, and an inline warning is shown. This is purely client-side — the REST endpoints that create and update content (`POST /o/c/forummessages`, `POST /o/c/forumreplies`, `PATCH /o/c/forummessages/{id}`, `PATCH /o/c/forumreplies/{id}`) have no knowledge of the `ForumBan` collection and will accept requests from a banned user if called directly.

**Why the legacy portlets don't have this gap:** The legacy Message Boards portlets enforce bans at the Liferay permission framework layer (`MBPortletResourcePermissionLogic`), which calls `MBBanLocalService.hasBan()` on every permission check regardless of the calling path (web UI, REST API, or direct service invocation). Custom Liferay Objects have no equivalent hook into that permission logic.

**Why this can't be fixed with built-in Object features:** Object Actions all fire after the entry is already committed (there is no pre-create trigger that can abort creation). Object Validation rules use the Expression Builder, which is limited to the entry's own field values and cannot query other Object collections or access current user context. Groovy script actions — which could perform the check — are not available on Liferay SaaS.

**The only realistic server-side option: Microservice Client Extension**

A Spring Boot Microservice Client Extension can be registered as an Object Action webhook on both `ForumMessage` and `ForumReply`, triggered on the `On After Add` event. It would:

1. Receive the Object Action payload, which includes the `creatorId` (the user ID of the entry author) and the `groupId` (site scope).
2. Call `GET /o/c/forumbans?filter=banUserId eq {creatorId}&pageSize=1` using service credentials to check for a ban record.
3. If a ban record exists, immediately call `DELETE /o/c/forummessages/{entryId}` or `DELETE /o/c/forumreplies/{entryId}` to remove the entry.

There is an unavoidable brief window (milliseconds to low seconds depending on load) between the entry being created and the microservice deleting it. In practice this is acceptable given that banning is rare and the moderation fragment provides a backstop for any content that appears during that window.

### No Endpoints for Discovering Subscribed Users

Research discovered a hard feature gap when building a Spring Boot Microservice Client Extension to power email/notification delivery for forum subscriptions — specifically, sending a notification to all users subscribed to a topic when a new reply is posted: **Liferay's headless REST APIs have no endpoint that returns which users are subscribed to a given Object entry (or Message Boards thread equivalent).**

The only subscription endpoints in the `headless-admin-user` API are scoped to the calling user: `GET /o/headless-admin-user/v1.0/my-user-account/subscriptions` returns the subscriptions belonging to the authenticated user making the request. There is no admin-facing endpoint that lists *all* subscribers for a resource. The underlying data lives in `SubscriptionLocalService` but is not exposed through any published REST API.

**Workaround 1 — "Forum Subscription" Object**

Introduce a new `ForumSubscription` Liferay Object with fields for `subscriberUserId`, `messageERC` (the subscribed topic), and `siteId`. When a user subscribes or unsubscribes, the fragment calls `POST` / `DELETE` on `/o/c/forumsubscriptions/` to maintain the record. The Spring Boot microservice (triggered by an Object Action on `ForumReply → On After Add`) then queries `GET /o/c/forumsubscriptions/?filter=messageERC eq '{erc}'` to obtain the full subscriber list and fans out the notifications. This is entirely within the Objects + headless stack and requires no portal-side code changes, but it means subscription state is owned by a custom Object rather than Liferay's native subscription infrastructure, and the two can drift if users subscribe through any other surface (e.g., via the legacy Message Boards portlet).

**Workaround 2 — REST Builder Endpoints**

Use Liferay's **REST Builder** code-generation tool (an OSGi module deployed to the portal) to generate a custom headless API that delegates to `SubscriptionLocalService`. A thin `GET /o/forum-subscriptions/v1.0/threads/{threadId}/subscribers` endpoint can call `SubscriptionLocalServiceUtil.getSubscriptions(companyId, ForumMessage.class.getName(), threadId)` server-side and return the subscriber user IDs or email addresses. The Spring Boot microservice then calls this custom endpoint instead of the missing platform one, keeping subscription state in Liferay's native store with no sync concerns. The trade-off is that REST Builder modules are traditional OSGi artifacts — not Client Extensions — so they cannot be deployed on Liferay SaaS and require a self-hosted or PaaS environment.

### "Top Replies" Implemented as "Recent Activity"

The "Recent Activity" tab (formerly "Top Replies") sorts messages using `lastPostDate:desc`. This functions as a "Recently Active" feed rather than filtering for the highest volume of total replies. A new message with 1 reply will surface above an older message with 100 replies.

**Option:** If a true "Top Replied" filter is desired, the sorting criteria must be changed to target a `replyCount` metric. The Liferay Object definition would need an aggregated integer field for total replies that can be passed to the OData `sort` parameter (e.g., `sort=replyCount:desc`), or rely on a Client Extension to dynamically aggregate and sort this information.
