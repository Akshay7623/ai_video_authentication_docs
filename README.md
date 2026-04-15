# ai_video_authentication_docs
Documentation for authentication flow of AI video Generation application


# 🔐 Exhaustive User Authentication API Reference

This document provides a definitive list of every possible response status code and payload for the authentication system.

## 🚀 Public Routes

### 1. Signup (`POST /api/auth/signup`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | OTP sent successfully (first time) | `{"message": "OTP has been sent.", "expiresIn": 120}` |
| **200** | OTP already active (duplicate request) | `{"message": "OTP already sent. Please verify.", "expiresIn": <rem>}` |
| **200** | Account exists but unverified (OTP expired) | `{"message": "OTP sent successfully.", "expiresIn": 120}` |
| **400** | Validation fail or invalid state | `{"message": "Invalid input", "errors": {...}}` |
| **409** | Email already registered & verified | `{"message": "User already registered. Please login."}` |
| **502** | Email sending service failure | `{"message": "Failed to send OTP email. Please retry."}` |
| **500** | Unexpected server error | `{"message": "Internal Server Error"}` |

---

### 2. Verify Signup OTP (`POST /api/auth/signup/verify-otp`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Verification successful | `{"message": "Email verified successfully.", "success": true, "accessToken": "...", "refreshToken": "..."}` |
| **400** | Missing/Invalid input format | `{"message": "Invalid input", "errors": {...}}` |
| **401** | Incorrect OTP digits | `{"message": "Invalid OTP. Please check and try again.", "success": false}` |
| **404** | Email not found in signup records | `{"message": "User not found. Please sign up first.", "success": false}` |
| **409** | Email is already active/verified | `{"message": "Email is already verified. Please log in.", "success": false}` |
| **410** | OTP expired (over 5 mins incl. grace) | `{"message": "OTP has expired. Please request a new one.", "success": false}` |
| **429** | Too many incorrect attempts (Max 5) | `{"message": "Maximum OTP attempts reached. Please request a new OTP.", "success": false}` |
| **500** | Unexpected server error | `{"message": "Internal server error", "success": false}` |

---

### 3. Login (`POST /api/auth/login`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Login successful | `{"accessToken": "...", "refreshToken": "..."}` |
| **400** | Missing email or password | `{"message": "Invalid input", "errors": {...}}` |
| **401** | Wrong password or email not found | `{"message": "Invalid credentials"}` |
| **401** | Account exists but email not verified | `{"message": "Please verify your email"}` |
| **423** | Account locked (Current request) | `{"message": "Account locked due to multiple failed login attempts..."}` |
| **423** | Account locked (Previous attempts) | `{"message": "Account is locked. Try again in X minute(s)."}` |
| **500** | Unexpected server error | `{"message": "Internal server error"}` |

---

### 4. Refresh Token (`POST /api/auth/refresh-token`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Rotation successful | `{"success": true, "accessToken": "...", "refreshToken": "..."}` |
| **400** | Invalid token string format | `{"success": false, "message": "Invalid input", "errors": {...}}` |
| **401** | Token expired or badly signed | `{"success": false, "message": "Refresh token has expired..."}` |
| **401** | Session invalidated/revoked | `{"success": false, "message": "No active session..."}` |
| **401** | Token already reused (Revocation) | `{"success": false, "message": "Refresh token is invalid or rotated."}` |
| **404** | User record missing | `{"success": false, "message": "User not found."}` |

---

### 5. Logout (`POST /api/auth/logout`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Success (Session ended) | `{"message": "Logged out successfully.", "success": true}` |
| **200** | Success (Token already dead) | `{"message": "Logged out successfully (session was already invalid).", "success": true}` |
| **400** | Body validation error | `{"message": "Invalid input", "errors": {...}}` |

---

### 6. Forgot Password (`POST /api/auth/forgot-password`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Recovery initiated (Success) | `{"message": "OTP sent successfully.", "expiresIn": 120}` |
| **200** | Email not found (Security mask) | `{"message": "User not found"}` |
| **200** | Cooldown (Wait 2 mins) | `{"message": "Please wait before requesting another OTP.", "expiresIn": <rem>}` |
| **400** | Bad email format | `{"message": "Invalid input data"}` |

---

### 7. Verify Reset OTP (`POST /api/auth/password/verify-otp`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Reset privilege granted | `{"message": "OTP verified successfully...", "success": true, "resetToken": "..."}` |
| **400** | Incorrect OTP or No OTP active | `{"message": "Invalid OTP..."}` OR `{"message": "No OTP requested..."}` |
| **404** | User not found/verified | `{"message": "No active password reset request..."}` |
| **410** | Reset OTP expired | `{"message": "Reset OTP has expired..."}` |
| **429** | Max attempts reached (Max 5) | `{"message": "Maximum OTP attempts reached..."}` |

---

### 8. Set New Password (`POST /api/auth/password/set-new-password`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Password updated | `{"message": "Password has been reset successfully...", "success": true}` |
| **401** | `resetToken` expired or invalid | `{"message": "Invalid or expired reset token...", "success": false}` |

---

## 🔒 Protected User Routes (Requires JWT)

> [!NOTE]
> All routes below require an `Authorization: Bearer <accessToken>` header.

### 9. Get Profile (`GET /api/user/profile`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Success | `{"success": true, "profile": { "email": "...", "fullName": "...", "role": "...", "status": "active", "createdAt": "..." } }` |
| **401** | Missing or invalid Access Token | `{"message": "Unauthorized missing identity context.", "success": false}` |

---

### 10. Change Password (`POST /api/auth/password/change`)
| Code | Condition | Payload |
| :--- | :--- | :--- |
| **200** | Credentials updated | `{"message": "Password changed successfully.", "success": true}` |
| **400** | New password too short (<8) | `{"message": "Invalid input", "errors": {...}}` |
| **401** | `oldPassword` is incorrect | `{"message": "Current password is incorrect.", "success": false}` |

---

## ⚙️ Generic Error States
Always applicable to all endpoints:
- **500 Internal Server Error**: `{"message": "Internal server error", "success": false}`
- **502 Bad Gateway**: `{"message": "Failed to send email. Please retry."}` (Email-dependent routes)
