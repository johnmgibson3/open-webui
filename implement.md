# Individual User Resource Sharing - Implementation Status

## Overview
Implementation of admin UI controls for configuring individual user sharing settings in Open WebUI.

## Changes Made

### Backend Changes

#### 1. Configuration (backend/open_webui/config.py)
**Lines: 1617-1633**
- Added 3 new `PersistentConfig` fields:
  - `ENABLE_INDIVIDUAL_USER_SHARING` (boolean, default: True)
  - `USER_SHARING_SCOPE` (string, default: "restricted")
  - `USER_SHARING_MODE` (string, default: "email")

```python
ENABLE_INDIVIDUAL_USER_SHARING = PersistentConfig(
    "ENABLE_INDIVIDUAL_USER_SHARING",
    "sharing.enable_individual_user_sharing",
    os.environ.get("ENABLE_INDIVIDUAL_USER_SHARING", "True").lower() == "true",
)

USER_SHARING_SCOPE = PersistentConfig(
    "USER_SHARING_SCOPE",
    "sharing.user_sharing_scope",
    os.environ.get("USER_SHARING_SCOPE", "restricted"),
)

USER_SHARING_MODE = PersistentConfig(
    "USER_SHARING_MODE",
    "sharing.user_sharing_mode",
    os.environ.get("USER_SHARING_MODE", "email"),
)
```

#### 2. Admin API Endpoints (backend/open_webui/routers/auths.py)

**Lines: 946-952** - GET endpoint returns sharing fields:
```python
return {
    # ... other fields ...
    "ENABLE_INDIVIDUAL_USER_SHARING": request.app.state.config.ENABLE_INDIVIDUAL_USER_SHARING,
    "USER_SHARING_SCOPE": request.app.state.config.USER_SHARING_SCOPE,
    "USER_SHARING_MODE": request.app.state.config.USER_SHARING_MODE,
}
```

**Lines: 955-976** - AdminConfig Pydantic model updated:
```python
class AdminConfig(BaseModel):
    # 15 REQUIRED fields
    SHOW_ADMIN_DETAILS: bool
    WEBUI_URL: str
    # ... other required fields ...

    # Optional sharing fields
    ENABLE_INDIVIDUAL_USER_SHARING: Optional[bool] = None
    USER_SHARING_SCOPE: Optional[str] = None
    USER_SHARING_MODE: Optional[str] = None
```

**Lines: 1023-1030** - POST endpoint handles sharing fields:
```python
if form_data.ENABLE_INDIVIDUAL_USER_SHARING is not None:
    request.app.state.config.ENABLE_INDIVIDUAL_USER_SHARING = form_data.ENABLE_INDIVIDUAL_USER_SHARING

if form_data.USER_SHARING_SCOPE is not None and form_data.USER_SHARING_SCOPE in ["restricted", "global"]:
    request.app.state.config.USER_SHARING_SCOPE = form_data.USER_SHARING_SCOPE

if form_data.USER_SHARING_MODE is not None and form_data.USER_SHARING_MODE in ["email", "search", "both"]:
    request.app.state.config.USER_SHARING_MODE = form_data.USER_SHARING_MODE
```

#### 3. Access Control Validation (backend/open_webui/routers/prompts.py & models.py)
**Lines: 19-26 (prompts.py), 44-51 (models.py)**
- Added `validate_access_control` function to strip user_ids when individual user sharing is disabled:
```python
def validate_access_control(access_control):
    """Remove user_ids from access_control if individual user sharing is disabled"""
    if not ENABLE_INDIVIDUAL_USER_SHARING and access_control:
        if isinstance(access_control, dict):
            for key in ['read', 'write']:
                if key in access_control and isinstance(access_control[key], dict):
                    access_control[key].pop('user_ids', None)
    return access_control
```
- Applied in create and update endpoints (lines 79-81, 149-151 in prompts.py)

### Frontend Changes

#### 1. New Sharing Settings Component
**File: src/lib/components/admin/Settings/Sharing.svelte (NEW)**
- Complete admin UI for managing sharing settings
- Three main controls:
  1. **Enable Individual User Sharing** (toggle switch)
  2. **Sharing Scope** (radio buttons: Restricted/Global)
  3. **User Selection Mode** (radio buttons: Email Only/Search Only/Both)

**Key Implementation Details:**
- Loads full `adminConfig` object on mount
- Binds directly to `adminConfig.FIELD_NAME` properties (critical for keeping all required fields)
- Initializes default values if fields are undefined
- Sends entire `adminConfig` object to API (prevents "Field required" errors)
- Null checks prevent saving before config loads:
  ```typescript
  const updateHandler = async () => {
      if (!adminConfig) {
          toast.error($i18n.t('Configuration not loaded yet'));
          return;
      }
      // ... rest of save logic
  };
  ```

#### 2. Settings Navigation (src/lib/components/admin/Settings.svelte)
**Lines: 12, 37-50, 117-140, 450-458**
- Added "Sharing" tab to admin settings navigation
- Added import: `import Sharing from './Settings/Sharing.svelte';`
- Added tab button with share icon
- Added conditional rendering for Sharing component

## Current Status

### ✅ Completed
1. Backend configuration fields created
2. Admin API endpoints updated to handle sharing settings
3. Access control validation implemented for prompts and models
4. Frontend Sharing settings UI component created
5. Settings navigation updated with Sharing tab
6. Fixed null adminConfig handling in Sharing.svelte
7. Docker image rebuilt with latest changes (`open-webui:sharing-config-fix`)
8. Container deployed at http://localhost:3000

### 🔄 In Progress
- Testing the UI to verify settings display and save correctly

### ⚠️ Known Issues Fixed
1. **Issue**: "Field required" error when saving sharing settings
   - **Cause**: Sending null or partial adminConfig object
   - **Fix**: Added null checks, disabled save button until config loads, bind directly to adminConfig properties

2. **Issue**: AttributeError: Config key 'ENABLE_INDIVIDUAL_USER_SHARING' not found
   - **Cause**: Docker image didn't have latest backend code
   - **Fix**: Rebuilt Docker image with all changes

## Testing Checklist

### Backend Testing
- [x] Config fields created in backend/open_webui/config.py
- [x] GET /api/v1/auths/admin/config returns sharing fields
- [x] POST /api/v1/auths/admin/config accepts sharing fields
- [x] validate_access_control function in prompts.py and models.py

### Frontend Testing (Needs Verification)
- [ ] Navigate to Admin → Settings → Sharing
- [ ] Verify all controls display correctly
- [ ] Toggle "Enable Individual User Sharing" on/off
- [ ] Change "Sharing Scope" between Restricted/Global
- [ ] Change "User Selection Mode" between Email Only/Search Only/Both
- [ ] Click Save and verify success toast
- [ ] Refresh page and verify settings persist
- [ ] Test with "Enable Individual User Sharing" OFF:
  - [ ] Verify Sharing Scope and User Selection Mode are hidden
  - [ ] Create a prompt/model with access control
  - [ ] Verify user_ids are stripped from access_control

### Integration Testing (Not Yet Started)
- [ ] Create a prompt with "Enable Individual User Sharing" ON
  - [ ] Verify Users field appears in access control
  - [ ] Add individual users to access control
  - [ ] Save and verify users are persisted
- [ ] Toggle "Enable Individual User Sharing" OFF
  - [ ] Edit existing prompt with user_ids in access control
  - [ ] Save and verify user_ids are removed
- [ ] Test USER_SHARING_SCOPE setting:
  - [ ] Set to "restricted" and verify user search is limited to group members
  - [ ] Set to "global" and verify user search includes all users
- [ ] Test USER_SHARING_MODE setting:
  - [ ] Set to "email" and verify only email input is shown
  - [ ] Set to "search" and verify only name search is shown
  - [ ] Set to "both" and verify both options are available

## Deployment

### Current Deployment
- **Docker Image**: `open-webui:sharing-config-fix`
- **Container**: `open-webui-admin-test`
- **Port**: http://localhost:3000
- **Status**: Running (health: starting)

### Build Command
```bash
docker build -t open-webui:sharing-config-fix .
```

### Run Command
```bash
docker stop open-webui-admin-test && docker rm open-webui-admin-test
docker run -d -p 3000:8080 --name open-webui-admin-test open-webui:sharing-config-fix
```

## Architecture Notes

### Critical Pattern: AdminConfig Handling
The AdminConfig Pydantic model has **15 REQUIRED fields** (no Optional, no defaults). This means:
1. Frontend must load the full config object
2. Frontend must send the entire config object back (not just changed fields)
3. All 15 required fields must always be present in POST requests
4. Sharing fields are Optional to maintain backward compatibility

### Why Direct Binding Matters
The Sharing.svelte component binds directly to `adminConfig.FIELD_NAME`:
```svelte
<Switch bind:state={adminConfig.ENABLE_INDIVIDUAL_USER_SHARING} />
```

This is critical because:
- It maintains all 15 required fields in the adminConfig object
- Changes are automatically reflected in the object
- Sending adminConfig directly includes all required fields
- Matches the pattern used in General.svelte (reference implementation)

### Access Control Validation
The `validate_access_control` function ensures data integrity:
- Only prompts.py and models.py have this function (intentional)
- Other resources (tools, knowledge, notes, functions) do NOT strip user_ids
- This is by design - only prompts and models support individual user sharing feature toggle
- Other resources always support individual user sharing regardless of the setting

## Next Steps

1. **Immediate**: Test the UI at http://localhost:3000
   - Log in as admin
   - Navigate to Admin → Settings → Sharing
   - Verify all controls work correctly

2. **If UI works**: Test integration with prompts/models
   - Create new prompt with access control
   - Toggle sharing settings and verify behavior

3. **If UI doesn't work**: Debug frontend
   - Check browser console for errors
   - Check Docker logs for backend errors
   - Verify adminConfig is loading correctly

4. **Production Ready**: After testing succeeds
   - Create pull request with all changes
   - Document feature in user guide
   - Add tests for validate_access_control function
