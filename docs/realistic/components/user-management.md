# User Management

## Description

The User Management module is the administrative interface for managing users within the Admin Dashboard. This module allows support operators and administrators to view, edit, and manage user accounts on the TechCorp platform.

The module offers complete user lifecycle management functionalities, from viewing registration data to advanced operations such as merging duplicate accounts, fraud blocking, and permission management.

Unlike the user-service which is the data backend, this module is focused on the administrative operator experience, offering optimized workflows for the most common customer support tasks.

## Owners

- **Team:** Internal Tools
- **Tech Lead:** Diego Martins
- **Slack:** #internal-admin

## Access

The module is available at: `https://admin.techcorp.com/users`

### Required Permissions

| Action | Minimum Profile |
|--------|-----------------|
| View user | support |
| Edit registration data | operations |
| Block/Unblock | operations |
| Reset password | support |
| Merge accounts | admin |
| Delete user | admin |

## Features

### 1. User Search

Search allows finding users by multiple criteria:

| Field | Search Type |
|-------|-------------|
| Email | Exact or partial |
| Tax ID | Exact (with or without formatting) |
| Phone | Exact or last 4 digits |
| Name | Partial (fuzzy) |
| Order ID | Finds associated user |

### 2. Profile View

The user profile displays:

- **Registration Data:** Name, email, phone, tax ID, date of birth
- **Status:** Active, Inactive, Blocked, Pending verification
- **Order History:** Last 10 orders with link to details
- **Addresses:** List of registered addresses
- **Preferences:** Notification and language settings
- **Timeline:** Account action history

### 3. Data Editing

Fields editable by the operator:

| Field | Validation |
|-------|------------|
| Name | Minimum 2 characters |
| Email | Valid format, unique in system |
| Phone | Valid format (+1...) |
| Date of birth | Valid date, over 18 years old |

**Note:** Email change requires user confirmation via link sent to the new address.

### 4. Account Blocking

Available blocking reasons:

- Fraud suspicion
- User request
- Delinquency
- Terms violation
- Other (requires description)

### 5. Password Reset

The operator can request a password reset, which sends a recovery link to the user's email. The operator does **not** have access to the current password nor can set a new password directly.

### 6. Account Merge

Process to unify duplicate accounts:

1. Select primary account (will be kept)
2. Select secondary account (will be deactivated)
3. Review data to be migrated:
   - Orders
   - Addresses
   - Preferences
   - Credit balance
4. Confirm operation (irreversible)

## Common Workflows

### Customer cannot access account

1. Search user by email or tax ID
2. Check account status
3. If blocked, check reason in timeline
4. If active, request password reset
5. Verify email is correct

### Fraud suspicion

1. Search user
2. Analyze order history
3. Check suspicious patterns:
   - Multiple delivery addresses
   - Frequent data changes
   - Previous chargebacks
4. If suspicion confirmed, block account with reason "Fraud suspicion"
5. Notify anti-fraud team

### GDPR request (data deletion)

1. Search user
2. Check if there are orders in progress
3. If yes, wait for completion
4. Export user data
5. Request deletion from admin
6. Confirm deletion after retention period

## Troubleshooting

### Issue: Search doesn't find user that exists

**Cause:** Inconsistent search data or outdated cache.

**Solution:**
1. Try search by different field (email, tax ID, phone)
2. Check if tax ID is formatted correctly
3. Clear filters and try again
4. Use user ID if available

### Issue: Cannot edit user data

**Cause:** Insufficient permission or account in special state.

**Solution:**
1. Check your access profile
2. Check if account is not in deletion process
3. Check if there is pending operation (merge, verification)
4. Request permission elevation if necessary

### Issue: Reset email not arriving

**Cause:** Email in spam, blocked, or bounce.

**Solution:**
1. Check if registered email is correct
2. Request customer to check spam folder
3. Check delivery status in notification-service
4. If bounce, request email update

### Issue: Account merge failed

**Cause:** Data conflict or processing error.

**Solution:**
1. Check error logs in panel
2. Check if both accounts exist
3. Check if there is no pending operation on either account
4. Try again or escalate to technical support

## Audit

All actions in the module are logged:

- User who performed the action
- Date and time
- Action type
- Data before and after (for edits)
- Source IP

To query audit history, access the "Audit" tab in the user profile or use the Logs module.

## Related Links

- [User Service](user-service.md) - User data backend
- [Auth Service](auth-service.md) - Authentication and password reset
- [Admin Dashboard](admin-dashboard.md) - Complete administrative panel
- [Users API](../apis/users-api.md) - API routes
