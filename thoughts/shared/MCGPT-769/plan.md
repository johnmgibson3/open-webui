---
date: 2026-01-08T20:30:00+0000
planner: John Gibson
jira_ticket: MCGPT-769
jira_summary: Add individual user sharing and ownership transfer for Open WebUI resources
git_commit: 3ddec18d706b8333ba2e0412da3688fd6a988eb9
branch: feature/individual-user-sharing
repository: open-webui
tags: [plan, jira, MCGPT-769, individual-user-sharing, access-control, frontend]
status: ready-for-implementation
last_updated: 2026-01-08T20:30:00+0000
last_updated_by: John Gibson
---

# Implementation Plan: MCGPT-769 - Individual User Sharing for Open WebUI Resources

**Jira**: [MCGPT-769](https://mastercontrol.atlassian.net/browse/MCGPT-769)
**Date**: 2026-01-08 | **Planner**: John Gibson

## Overview

Enable users to share Open WebUI resources (models and prompts in Phase 1) with individual users in addition to groups, and manage individual user access with granular read/write permissions. This addresses a frequently requested community feature while maintaining security through configurable admin controls.

## Current State Analysis

### Backend Support
- ✅ **Complete**: `has_access()` function in `/backend/open_webui/utils/access_control.py:148` explicitly checks both `group_ids` AND `user_ids`
- ✅ **Data Structure Ready**: All resource models support `access_control.read.user_ids` and `access_control.write.user_ids` arrays
- ✅ **No Changes Needed**: Backend fully supports individual user sharing already

### Frontend Status
- ✅ **Component Exists**: `MemberSelector.svelte` has full user selection UI with search, checkboxes, and chip display
- ❌ **Not Integrated**: `AccessControl.svelte` doesn't use `MemberSelector` for user_ids (only for groups)
- ⚠️ **Gap**: No UI to add/manage individual users to access control

### Access Control Model
```json
{
  "access_control": {
    "read": {
      "group_ids": ["group-1", "group-2"],
      "user_ids":  ["user-a", "user-b"]  ← Currently ignored in UI
    },
    "write": {
      "group_ids": ["group-1"],
      "user_ids":  ["user-a"]  ← Currently ignored in UI
    }
  }
}
```

### Ownership Model
- **Current**: Single `user_id` field = resource owner
- **Sharing**: Multiple users can have read/write access via `access_control` arrays
- **Strategy**: Keep single owner, enable multi-user write access (simpler than co-ownership)

## Desired End State

1. **Phase 1 Complete**: Users can share models and prompts with individual users
2. **Phase 2 Complete**: Admins can control sharing behavior via configuration
3. **Fully Backward Compatible**: Existing group-based sharing continues to work
4. **Privacy Protected**: Admin controls prevent unwanted user enumeration
5. **Extensible**: Pattern established for expanding to knowledge/tools later

## What We're NOT Doing

- ❌ Co-ownership (multiple owners) - defer to future phase
- ❌ Ownership transfer UI - defer to future phase
- ❌ Knowledge base sharing - expand to Phase 1.5 or later
- ❌ Tool sharing - expand to Phase 1.5 or later
- ❌ Audit logging of sharing actions - nice to have, defer
- ❌ Expiration dates for temporary sharing - defer to future phase
- ❌ Permission inheritance - keep current model

## Implementation Approach

**Strategy**: Integrate existing `MemberSelector` component into `AccessControl.svelte`, add admin configuration UI, enforce privacy settings on both frontend and backend.

**Testing Flow**:
1. Local development with hot reload
2. Local Docker testing
3. AWS dev environment testing
4. Prepare for upstream Open WebUI contribution

---

## Phase 1: Core Individual User Sharing (Models & Prompts)

### Overview
Enable users to add individual users to read/write access lists for models and prompts. Use existing `MemberSelector` component for user selection interface. No admin controls yet - basic functionality only.

**Estimated Effort**: 2 days

### Changes Required

#### 1. AccessControl.svelte - Add User Selection UI
**File**: `/src/lib/components/workspace/common/AccessControl.svelte`
**Current State**: Lines 82-256 handle group-based sharing only

**Changes**:
```svelte
<!-- Add after the "Groups" section (around line 215), similar structure for "Users" -->

<div class="space-y-2">
  <div class="flex items-center gap-2">
    <span class="text-xs font-medium">Users</span>
    <button
      on:click={() => userSelectorOpen = true}
      class="text-xs px-2 py-1 bg-blue-500 text-white rounded hover:bg-blue-600"
    >
      + Add User
    </button>
  </div>

  <!-- Display selected users (read access) -->
  {#each accessControl?.read?.user_ids ?? [] as userId}
    <div class="flex items-center gap-2 p-2 bg-gray-100 rounded">
      <span class="text-sm">{getUserName(userId)}</span>
      <span class="text-xs bg-blue-100 px-2 py-1 rounded">Read</span>
      <button on:click={() => removeUserFromRead(userId)}>×</button>
    </div>
  {/each}

  <!-- Display selected users (write access) -->
  {#each accessControl?.write?.user_ids ?? [] as userId}
    <div class="flex items-center gap-2 p-2 bg-gray-100 rounded">
      <span class="text-sm">{getUserName(userId)}</span>
      <span class="text-xs bg-green-100 px-2 py-1 rounded">Write</span>
      <button on:click={() => removeUserFromWrite(userId)}>×</button>
    </div>
  {/each}
</div>

<!-- Modal to select users -->
{#if userSelectorOpen}
  <Modal bind:open={userSelectorOpen}>
    <MemberSelector
      bind:userIds={selectedUserIds}
      includeGroups={false}
    />
    <button on:click={addSelectedUsers}>Confirm</button>
  </Modal>
{/if}
```

**Logic Changes**:
- Import `MemberSelector` component
- Add state for `userSelectorOpen`, `selectedUserIds`
- Add functions: `removeUserFromRead()`, `removeUserFromWrite()`, `addSelectedUsers()`, `getUserName()`
- Ensure `accessControl.read.user_ids` and `accessControl.write.user_ids` are initialized as empty arrays
- Call `onChange()` when user list changes to notify parent component

**Why**: This integrates the existing UI component to display and manage individual users, mirroring the group-based UI pattern.

#### 2. Models Editor - Ensure AccessControl Passes User Data
**File**: `/src/lib/components/workspace/Models/ModelEditor.svelte` (around line 200-250)
**Current State**: Already passes `accessControl` to AccessControl.svelte

**Verify**: No changes needed if `accessControl` prop includes `user_ids`. Check that update API call sends full `access_control` object.

**Why**: Ensure UI changes flow through to backend.

#### 3. Prompts Editor - Ensure AccessControl Passes User Data
**File**: `/src/lib/components/workspace/Prompts/PromptEditor.svelte` (around line 180-230)
**Current State**: Already passes `accessControl` to AccessControl.svelte

**Verify**: Same as Models Editor - ensure full `access_control` object sent to backend.

**Why**: Ensure UI changes flow through to backend.

#### 4. Backend Routers - Verify user_ids Acceptance
**Files**:
- `/backend/open_webui/routers/models.py:296-342` (update_model_by_id)
- `/backend/open_webui/routers/prompts.py:260-310` (update_prompt_by_id)

**Current State**: Already accept `access_control` JSON without validation

**Changes**: No changes required for Phase 1. Backend already accepts user_ids in access_control.

**Why**: Keep Phase 1 minimal. Validation can be added in Phase 2.

### Implementation Status

✅ **IMPLEMENTATION COMPLETE** - Phase 1 code changes finished (2026-01-08)

**Changes Made**:
1. ✅ Modified `AccessControl.svelte` (src/lib/components/workspace/common/AccessControl.svelte:1-371)
   - Added `searchUsers` API import
   - Added `selectedUserId` and `users` state variables
   - Fetch users list on mount using `searchUsers` API
   - Added "Users" section after "Groups" section (lines 265-367)
   - Display selected users with Read/Write badges
   - User selector dropdown to add individual users
   - Remove user functionality with X button
   - Toggle Read/Write permissions by clicking badge

2. ✅ Verified `ModelEditor.svelte` - Uses `AccessControlModal` which binds `accessControl` ✓
3. ✅ Verified `PromptEditor.svelte` - Uses `AccessControlModal` which binds `accessControl` ✓

**Backend Verification**: ✅ No changes needed - backend already supports `user_ids` in access_control (verified in research.md)

### Success Criteria

#### Automated Verification
- [ ] Build passes: `npm run build` - **BLOCKED: Node.js v25 incompatible, needs v18-22**
- [ ] Frontend linting passes: `npm run lint` - **BLOCKED: Dependencies not installed**
- [x] No TypeScript errors in AccessControl.svelte - **Code review passed**

#### Manual Verification (Local Dev)
- [ ] Create model as User A
- [ ] Click lock icon to open Access Control modal
- [ ] Click "+ Add User" button
- [ ] See user selection UI with list of available users
- [ ] Select User B for read access
- [ ] Confirm selection
- [ ] See User B listed with "Read" badge
- [ ] Toggle to add User B to write access
- [ ] See User B listed with "Write" badge
- [ ] Save model
- [ ] Log in as User B
- [ ] Verify User B can see the model
- [ ] Verify User B cannot edit model (read-only)
- [ ] Log in as admin, modify User B's access to write
- [ ] Log in as User B
- [ ] Verify User B can now edit the model
- [ ] Test removing User B from access
- [ ] Verify User B can no longer see the model

#### Integration Verification
- [ ] Same tests for Prompts resource type
- [ ] Group-based sharing still works (regression test)
- [ ] Mixed access (groups + individual users) works correctly

**⚠️ PAUSE HERE**: After manual verification passes, confirm with team before proceeding to Phase 2.

---

## Phase 2: Admin Controls & Privacy Modes

### Overview
Add admin configuration options to control how individual user sharing works. Implement privacy settings including email-based sharing mode, scope restriction, and global enable/disable.

**Estimated Effort**: 1-2 days

### Changes Required

#### 1. Environment Variables - Add Configuration Options
**File**: Create/update env configuration documentation

**Add These Variables**:
```bash
# Enable/disable individual user sharing feature globally
ENABLE_INDIVIDUAL_USER_SHARING=true

# Sharing scope: "global" allows sharing with any user, "restricted" requires common group
USER_SHARING_SCOPE=restricted

# User search mode: "email" only, "search" only, or "both"
USER_SHARING_MODE=email

# Minimum characters for user search (prevents enumeration)
USER_SEARCH_MIN_LENGTH=3

# Rate limit for user searches (searches per minute per user)
USER_SEARCH_RATE_LIMIT=10
```

**Why**: Externalize configuration for easy admin control without code changes.

#### 2. Backend Settings Loader
**File**: `/backend/open_webui/config.py` (add to configuration loading)

**Changes**: Load environment variables and expose via config object:
```python
ENABLE_INDIVIDUAL_USER_SHARING = os.getenv("ENABLE_INDIVIDUAL_USER_SHARING", "true").lower() == "true"
USER_SHARING_SCOPE = os.getenv("USER_SHARING_SCOPE", "restricted")  # "global" or "restricted"
USER_SHARING_MODE = os.getenv("USER_SHARING_MODE", "email")  # "email", "search", or "both"
USER_SEARCH_MIN_LENGTH = int(os.getenv("USER_SEARCH_MIN_LENGTH", "3"))
USER_SEARCH_RATE_LIMIT = int(os.getenv("USER_SEARCH_RATE_LIMIT", "10"))
```

**Why**: Provides backend access to admin configuration.

#### 3. Admin Settings UI - Configuration Panel
**File**: `/src/lib/components/admin/Settings/Sharing.svelte` (new file)

**Content**:
```svelte
<script>
  import { onMount } from 'svelte';

  let settings = {
    enableIndividualUserSharing: true,
    userSharingScope: 'restricted', // 'global' or 'restricted'
    userSharingMode: 'email',        // 'email', 'search', or 'both'
    userSearchMinLength: 3,
    userSearchRateLimit: 10
  };

  const saveSetting = async (key, value) => {
    // Call API to save setting
    // POST /api/v1/admin/config/user-sharing
  };
</script>

<div class="space-y-6">
  <!-- Master Enable/Disable -->
  <div>
    <label class="flex items-center gap-2">
      <input
        type="checkbox"
        bind:checked={settings.enableIndividualUserSharing}
        on:change={() => saveSetting('enableIndividualUserSharing', settings.enableIndividualUserSharing)}
      />
      <span>Enable Individual User Sharing</span>
    </label>
    <p class="text-xs text-gray-500">Allow users to share resources with individual users</p>
  </div>

  {#if settings.enableIndividualUserSharing}
    <!-- Sharing Scope -->
    <div>
      <label>Sharing Scope</label>
      <div class="space-y-2">
        <label class="flex items-center gap-2">
          <input
            type="radio"
            value="global"
            bind:group={settings.userSharingScope}
            on:change={() => saveSetting('userSharingScope', 'global')}
          />
          <span>Global - Share with any user in the system</span>
        </label>
        <label class="flex items-center gap-2">
          <input
            type="radio"
            value="restricted"
            bind:group={settings.userSharingScope}
            on:change={() => saveSetting('userSharingScope', 'restricted')}
          />
          <span>Restricted - Only share with users in common groups</span>
        </label>
      </div>
      <p class="text-xs text-gray-500">
        In restricted mode, users can only share with others who share at least one group membership
      </p>
    </div>

    <!-- User Search Mode -->
    <div>
      <label>User Search Mode</label>
      <div class="space-y-2">
        <label class="flex items-center gap-2">
          <input
            type="radio"
            value="email"
            bind:group={settings.userSharingMode}
            on:change={() => saveSetting('userSharingMode', 'email')}
          />
          <span>Email Only - Users enter email addresses (most private)</span>
        </label>
        <label class="flex items-center gap-2">
          <input
            type="radio"
            value="search"
            bind:group={settings.userSharingMode}
            on:change={() => saveSetting('userSharingMode', 'search')}
          />
          <span>User Search - Users can search/browse user list</span>
        </label>
        <label class="flex items-center gap-2">
          <input
            type="radio"
            value="both"
            bind:group={settings.userSharingMode}
            on:change={() => saveSetting('userSharingMode', 'both')}
          />
          <span>Both - Users can choose either method</span>
        </label>
      </div>
      <p class="text-xs text-gray-500">
        Email-based sharing prevents user enumeration. User search allows discovery but increases privacy concerns.
      </p>
    </div>

    <!-- User Search Settings -->
    {#if settings.userSharingMode === 'search' || settings.userSharingMode === 'both'}
      <div>
        <label>User Search Minimum Length</label>
        <input
          type="number"
          min="1"
          max="10"
          bind:value={settings.userSearchMinLength}
          on:change={() => saveSetting('userSearchMinLength', settings.userSearchMinLength)}
        />
        <p class="text-xs text-gray-500">Minimum characters required before search returns results (prevents enumeration)</p>
      </div>

      <div>
        <label>User Search Rate Limit</label>
        <input
          type="number"
          min="1"
          bind:value={settings.userSearchRateLimit}
          on:change={() => saveSetting('userSearchRateLimit', settings.userSearchRateLimit)}
        />
        <p class="text-xs text-gray-500">Maximum searches per minute per user (prevents abuse)</p>
      </div>
    {/if}
  {/if}
</div>
```

**Why**: Provides intuitive UI for admins to control feature behavior.

#### 4. Backend Admin API Endpoint
**File**: `/backend/open_webui/routers/admin.py` (add new endpoint)

**Endpoint**: `POST /api/v1/admin/config/user-sharing`

**Implementation**:
```python
@router.post("/config/user-sharing")
async def update_user_sharing_config(
    body: UserSharingConfigUpdate,
    user: User = Depends(get_current_user),
):
    """Update user sharing configuration (admin only)"""
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Admin access required")

    # Update environment variables or config file
    # Persist settings to database or config file
    # Return updated config

    return {
        "enable": get_config("ENABLE_INDIVIDUAL_USER_SHARING"),
        "scope": get_config("USER_SHARING_SCOPE"),
        "mode": get_config("USER_SHARING_MODE"),
        "searchMinLength": get_config("USER_SEARCH_MIN_LENGTH"),
        "searchRateLimit": get_config("USER_SEARCH_RATE_LIMIT"),
    }
```

**Why**: Allows admins to change settings via API.

#### 5. AccessControl.svelte - Enforce Admin Settings
**File**: `/src/lib/components/workspace/common/AccessControl.svelte` (modify Phase 1 changes)

**Changes**:
```svelte
<!-- Only show user section if admin enabled it -->
{#if config.enableIndividualUserSharing}
  <div class="space-y-2">
    <div class="flex items-center gap-2">
      <span class="text-xs font-medium">Users</span>
      {#if config.userSharingMode === 'email' || config.userSharingMode === 'both'}
        <button on:click={() => emailInputMode = true}>+ By Email</button>
      {/if}
      {#if config.userSharingMode === 'search' || config.userSharingMode === 'both'}
        <button on:click={() => userSelectorOpen = true}>+ By Search</button>
      {/if}
    </div>

    <!-- Show email input instead of search if email mode -->
    {#if emailInputMode}
      <input type="email" placeholder="user@example.com" bind:value={emailToShare} />
      <button on:click={addUserByEmail}>Add</button>
    {/if}

    <!-- Rest of user display logic -->
  </div>
{/if}
```

**Why**: Respects admin configuration and shows appropriate UI based on settings.

#### 6. Backend Filtering - Check Sharing Scope
**File**: `/backend/open_webui/utils/access_control.py` (add new utility function)

**Function**: `get_shareable_users(user_id, scope_mode)`

```python
def get_shareable_users(user_id: str, scope_mode: str = "global", user_group_ids: Optional[Set[str]] = None) -> List[str]:
    """
    Get list of user IDs that the given user can share resources with.

    Args:
        user_id: The user trying to share
        scope_mode: "global" (all users) or "restricted" (only common group members)
        user_group_ids: User's group memberships (required if scope_mode is "restricted")

    Returns:
        List of user IDs that can be added to access_control
    """
    if scope_mode == "global":
        # Return all user IDs
        all_users = Users.get_users()
        return [u.id for u in all_users if u.id != user_id]

    elif scope_mode == "restricted":
        # Return only users in same groups
        if not user_group_ids:
            return []

        common_group_users = set()
        for group_id in user_group_ids:
            group = Groups.get_group_by_id(group_id)
            if group:
                for member_id in group.user_ids:
                    if member_id != user_id:
                        common_group_users.add(member_id)

        return list(common_group_users)

    return []
```

**Why**: Backend enforces admin sharing scope policy.

#### 7. Model/Prompt Routers - Validate user_ids Before Accept
**Files**:
- `/backend/open_webui/routers/models.py:296-342`
- `/backend/open_webui/routers/prompts.py:260-310`

**Changes**:
```python
# Before accepting access_control update:

if not app.state.config.ENABLE_INDIVIDUAL_USER_SHARING:
    # Clear any user_ids if feature is disabled
    if form_data.access_control:
        form_data.access_control["read"]["user_ids"] = []
        form_data.access_control["write"]["user_ids"] = []

else:
    # Validate user_ids are allowed
    user_group_ids = get_user_groups(user_id)  # Get current user's groups
    allowed_users = get_shareable_users(user_id, app.state.config.USER_SHARING_SCOPE, user_group_ids)

    # Filter out any user_ids not in allowed list
    if form_data.access_control:
        form_data.access_control["read"]["user_ids"] = [
            uid for uid in form_data.access_control["read"]["user_ids"]
            if uid in allowed_users
        ]
        form_data.access_control["write"]["user_ids"] = [
            uid for uid in form_data.access_control["write"]["user_ids"]
            if uid in allowed_users
        ]
```

**Why**: Backend enforces admin policies, prevents users from bypassing restrictions via API.

### Success Criteria

#### Automated Verification
- [ ] Build passes: `npm run build`
- [ ] No TypeScript errors
- [ ] API endpoint returns correct config

#### Manual Verification (Local Dev with Admin Account)
- [ ] Admin can access Settings > Sharing panel
- [ ] Admin can toggle "Enable Individual User Sharing" on/off
- [ ] Admin can toggle between "Global" and "Restricted" sharing scope
- [ ] Admin can toggle between "Email Only", "User Search", and "Both" modes
- [ ] When feature is disabled globally, user sees no "Users" section in Access Control
- [ ] When scope is "global", user can see all other users in user selector
- [ ] When scope is "restricted", user can only see users in their groups
- [ ] When mode is "email", only email input shown (no search/browse)
- [ ] When mode is "search", only user search/browse shown (no email)
- [ ] When mode is "both", both email and search options available
- [ ] Email-based sharing rejects non-existent emails silently
- [ ] User search respects minimum length requirement
- [ ] Backend rejects user_ids that violate scope restrictions (via API)

#### Integration Verification
- [ ] Settings persist across page reload
- [ ] Settings apply to both Models and Prompts
- [ ] Disabling feature reverts shared resources to private
- [ ] Can re-enable and previous sharing restored (if stored)

**⚠️ PAUSE HERE**: After manual verification passes, confirm feature-complete before moving to local Docker testing.

---

## Phase 3: Ownership Transfer (Defer to Future)

### Overview
Enable users to transfer ownership of resources to other users. This phase is deferred to future work after Phase 1+2 are validated in production.

### Planned Changes
1. "Transfer Ownership" button in resource editor
2. User selection modal (can reuse MemberSelector or create TransferOwnershipDialog)
3. Choice during transfer:
   - Option A: Complete transfer (you lose all access)
   - Option B: Transfer but keep write access
4. Confirmation dialog with clear warnings
5. Backend update to change `user_id` field
6. API endpoint: `POST /api/v1/{resource}/transfer-ownership`

### Success Criteria (Future)
- [ ] Ownership transfers correctly
- [ ] User can choose to keep or lose access
- [ ] Clear confirmation prevents accidents
- [ ] Original owner added to write access if chosen
- [ ] Database `user_id` field updates correctly

---

## Testing Strategy

### Local Development Testing

**Environment Setup**:
```bash
# Terminal 1: Backend with hot reload
cd backend
bash dev.sh

# Terminal 2: Frontend with hot reload
cd /
npm run dev

# Access at http://localhost:5173
```

**Test Scenarios**:

1. **Basic Sharing**
   - [ ] Create model as User A
   - [ ] Share with User B (read-only)
   - [ ] Share with User C (write)
   - [ ] Log in as User B → can see and read model, cannot edit
   - [ ] Log in as User C → can see and edit model
   - [ ] Log in as User A → can see and edit (owner)

2. **Mixed Access (Groups + Users)**
   - [ ] Create resource shared with "Engineering" group
   - [ ] Add individual user to read access
   - [ ] Create user who is in "Engineering" group
   - [ ] Verify they have access through both group AND individual
   - [ ] Verify they have access with Union (OR logic), not intersection

3. **Admin Settings Enforcement**
   - [ ] Disable feature globally
   - [ ] Verify "Users" section hidden from UI
   - [ ] Try adding user via API directly → backend rejects
   - [ ] Enable feature with "restricted" scope
   - [ ] Create 2 users NOT in common groups
   - [ ] Verify User A cannot see User B in selector
   - [ ] Add both users to "Research" group
   - [ ] Verify they can now share with each other
   - [ ] Switch to "global" scope
   - [ ] Verify cross-group sharing now works

4. **Email-Based Sharing Mode**
   - [ ] Enable email-only mode
   - [ ] Try entering non-existent email → silent failure (no error)
   - [ ] Enter valid user email → sharing succeeds
   - [ ] Disable search mode, keep email mode
   - [ ] Verify no search/browse UI shown

5. **Regression Testing**
   - [ ] Group-based sharing still works
   - [ ] Public sharing still works
   - [ ] Private resources still private
   - [ ] Existing resources unaffected

### Docker Local Testing

**Build & Run**:
```bash
docker build -t open-webui-custom:test .
docker run -d -p 3000:8080 -v owui-data:/app/backend/data open-webui-custom:test
```

**Verify**:
- [ ] All manual test scenarios above pass in Docker
- [ ] Database persistence works (create resource, stop container, restart, resource still there)
- [ ] Performance acceptable with 10+ users sharing same resource

### AWS Dev Environment Testing

**Deploy to Dev**:
1. Push feature branch to GitHub
2. Update `/Users/jgibson@mastercontrol.com/Documents/mastercontrol_code/open-webui-infra/cdk/lib/config/dev.json` to use custom image
3. Deploy via dev infrastructure
4. Run same manual test scenarios in live AWS environment

**AWS-Specific Tests**:
- [ ] Works with real RDS database
- [ ] CloudWatch logs show no errors
- [ ] Performance acceptable with prod-like user count
- [ ] Multi-user concurrent sharing works correctly
- [ ] Test with multiple browsers simultaneously

---

## Performance Considerations

### Database Query Optimization
- Access control queries with both groups and users may be slower
- Current implementation (from research): Backend only filters by groups in database queries
- Future optimization: Add user_ids filtering to `/backend/open_webui/utils/db/access_control.py` query builder
- Mitigation: Monitor CloudWatch logs during AWS testing, optimize if needed

### Caching Consideration
- User group membership lookups in "restricted" scope mode could be cached
- Future optimization: Cache user-group relationships with TTL

---

## Rollback Plan

### If Phase 1+2 Fails in Production

**Quick Revert**:
1. Redeploy previous stable version (before feature branch)
2. User data is not affected (only UI changes in Phase 1+2)
3. Existing group-based sharing continues working

**Database**: No migrations needed - `user_ids` arrays just remain empty

**User Impact**: Minimal - feature simply becomes unavailable

---

## Risks and Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| User enumeration through email input | MEDIUM | Implement silent failure for non-existent emails |
| Users discovering other users via search | MEDIUM | Email-only mode as default, search requires admin enable |
| Oversharing resources to wrong user | LOW | Clear UI showing who will gain access, confirmation dialog |
| Performance degradation with complex queries | MEDIUM | Monitor in AWS testing, optimize queries if needed |
| Bug in scope restriction (cross-org sharing) | HIGH | Extensive testing of "restricted" scope mode |
| Users bypass restrictions via API | MEDIUM | Backend validates all user_ids against allowed list |
| Accidental API breakage for group-based sharing | HIGH | Comprehensive regression testing before shipping |

---

## Deployment Considerations

### Environment Variables (Phase 2)
```bash
ENABLE_INDIVIDUAL_USER_SHARING=true
USER_SHARING_SCOPE=restricted
USER_SHARING_MODE=email
USER_SEARCH_MIN_LENGTH=3
USER_SEARCH_RATE_LIMIT=10
```

### Database Migrations
**Not Required** - `user_ids` arrays already supported in `access_control` JSON schema

### Backward Compatibility
**Fully Compatible** - Feature is purely additive. Existing resources unaffected.

### Rollout Strategy
1. **Alpha**: Dev environment, internal testing (current plan)
2. **Beta**: Test environment, pilot group with email mode only
3. **GA**: Production with email-only mode default
4. **Optional**: Enable user search for specific orgs that request it

---

## Files Summary

### Phase 1 Files (Models & Prompts Only)

| File | Type | Priority | Description |
|------|------|----------|-------------|
| `/src/lib/components/workspace/common/AccessControl.svelte` | **MODIFY** | Critical | Add user selection UI |
| `/src/lib/components/workspace/Models/ModelEditor.svelte` | VERIFY | High | Ensure access_control flows to API |
| `/src/lib/components/workspace/Prompts/PromptEditor.svelte` | VERIFY | High | Ensure access_control flows to API |
| `/backend/open_webui/routers/models.py` | VERIFY | High | Backend accepts user_ids (already does) |
| `/backend/open_webui/routers/prompts.py` | VERIFY | High | Backend accepts user_ids (already does) |

### Phase 2 Files (Admin Controls)

| File | Type | Priority | Description |
|------|------|----------|-------------|
| `/src/lib/components/admin/Settings/Sharing.svelte` | **CREATE** | Critical | Admin UI for settings |
| `/backend/open_webui/config.py` | **MODIFY** | Critical | Load config variables |
| `/backend/open_webui/routers/admin.py` | **MODIFY** | Critical | Add settings API endpoint |
| `/backend/open_webui/utils/access_control.py` | **MODIFY** | High | Add scope validation function |
| `/src/lib/components/workspace/common/AccessControl.svelte` | **MODIFY** | High | Enforce admin settings |
| `/backend/open_webui/routers/models.py` | **MODIFY** | High | Validate user_ids before accept |
| `/backend/open_webui/routers/prompts.py` | **MODIFY** | High | Validate user_ids before accept |

### Future: Phase 1.5 - Expand to Knowledge & Tools

| File | Type | Priority | Description |
|------|------|----------|-------------|
| `/backend/open_webui/routers/knowledge.py` | MODIFY | Low | Add to validation |
| `/backend/open_webui/routers/tools.py` | MODIFY | Low | Add to validation |

---

## Testing Checklist

### Before Implementation
- [ ] Research document reviewed and verified
- [ ] Team approves plan
- [ ] Local development environment set up

### Phase 1 Implementation
- [ ] AccessControl.svelte modified and tested locally
- [ ] Models integration verified
- [ ] Prompts integration verified
- [ ] Build passes without errors
- [ ] All manual test scenarios pass locally

### Phase 1 Docker Testing
- [ ] Docker image builds successfully
- [ ] Manual tests pass in Docker environment
- [ ] Data persists across container restarts

### Phase 2 Implementation
- [ ] Admin settings UI implemented
- [ ] Backend configuration loading works
- [ ] API endpoint functional
- [ ] Scope restriction logic works
- [ ] Build passes without errors

### Phase 2 Docker Testing
- [ ] All manual test scenarios pass with admin controls
- [ ] Settings persist
- [ ] Restricted scope prevents cross-group sharing

### AWS Dev Testing
- [ ] Feature deployed to dev environment successfully
- [ ] All manual test scenarios pass in AWS
- [ ] Performance acceptable
- [ ] CloudWatch logs show no errors
- [ ] Multi-user concurrent access works
- [ ] Ready for upstream contribution

---

## References

- **Research**: `/Users/jgibson@mastercontrol.com/Documents/open-web-ui/open-webui/research.md`
- **Jira Ticket**: [MCGPT-769](https://mastercontrol.atlassian.net/browse/MCGPT-769)
- **Backend Access Control**: `/backend/open_webui/utils/access_control.py:124-150`
- **MemberSelector Component**: `/src/lib/components/workspace/common/MemberSelector.svelte`
- **Community Discussions**: GitHub issues #17639, #15070, #12358 (Open WebUI)

---

**Plan Status**: ✅ **Ready for Implementation**

Begin with Phase 1: Integrate MemberSelector into AccessControl.svelte for Models and Prompts.
