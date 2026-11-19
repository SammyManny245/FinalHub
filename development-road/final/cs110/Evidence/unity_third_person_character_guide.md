# Unity 3rd-Person Character Setup & Troubleshooting Guide

This document captures the step-by-step process and code used to create, program, and debug a 3rd-person character in Unity.

---

## 1. Visual Overview & Setup Steps

![How to Create & Program a 3rd Person Character in Unity](watermarked_img_1450997389644422483.jpg)

### Key Setup Steps:
1. **Choose a Character Model:** Import a 3D character mesh (or a simple 3D Capsule) into your **Hierarchy** and place it on one of your platforms.
2. **Add Physics (Character Controller):** Select your player object, click **Add Component** in the Inspector, and add a `Character Controller` component to manage collision and movement.
3. **Setup Cinemachine Camera:** Install **Cinemachine** via `Window > Package Manager`. Add a **Cinemachine Virtual Camera** to the scene and set your Player object into both the `Follow` and `Look At` target slots.
4. **Program Player Movement:** Attach the script below (`PlayerController.cs`) to your Player object.
5. **Assign & Test:** Connect the script components in the Inspector, enter **Play Mode**, and test moving and jumping across platforms using **WASD / Arrow Keys** and **Spacebar**.

---

## 2. Complete C# Script (`PlayerController.cs`)

This script supports:
* **Unity New Input System** (`UnityEngine.InputSystem`)
* **Camera-relative movement**
* **Smooth rotation toward movement direction**
* **Gravity and jumping**

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

[RequireComponent(typeof(CharacterController))]
public class PlayerController : MonoBehaviour
{
    [Header("Movement Settings")]
    public float moveSpeed = 6f;
    public float rotationSmoothTime = 0.12f;

    [Header("Jump & Gravity")]
    public float jumpHeight = 2f;
    public float gravity = -20f;
    public float groundedGravity = -2f;

    [Header("Camera Reference")]
    public Transform cameraTransform;

    private CharacterController controller;
    private float verticalVelocity;
    private float rotationVelocity;

    private void Awake()
    {
        controller = GetComponent<CharacterController>();

        if (cameraTransform == null && Camera.main != null)
        {
            cameraTransform = Camera.main.transform;
        }
    }

    private void Update()
    {
        HandleMovementAndJump();
    }

    private void HandleMovementAndJump()
    {
        bool isGrounded = controller.isGrounded;

        // Reset downward velocity slightly when grounded so isGrounded stays reliable
        if (isGrounded && verticalVelocity < 0f)
        {
            verticalVelocity = groundedGravity;
        }

        float horizontal = 0f;
        float vertical = 0f;
        bool jumpPressed = false;

        Keyboard keyboard = Keyboard.current;
        if (keyboard != null)
        {
            if (keyboard.aKey.isPressed || keyboard.leftArrowKey.isPressed)
                horizontal -= 1f;
            if (keyboard.dKey.isPressed || keyboard.rightArrowKey.isPressed)
                horizontal += 1f;

            if (keyboard.sKey.isPressed || keyboard.downArrowKey.isPressed)
                vertical -= 1f;
            if (keyboard.wKey.isPressed || keyboard.upArrowKey.isPressed)
                vertical += 1f;

            jumpPressed = keyboard.spaceKey.wasPressedThisFrame;
        }

        Vector3 inputDirection = new Vector3(horizontal, 0f, vertical).normalized;

        // Calculate camera-relative horizontal movement
        Vector3 moveDirection = Vector3.zero;
        if (inputDirection.magnitude >= 0.1f)
        {
            float targetAngle = Mathf.Atan2(inputDirection.x, inputDirection.z) * Mathf.Rad2Deg;

            if (cameraTransform != null)
            {
                targetAngle += cameraTransform.eulerAngles.y;
            }

            float angle = Mathf.SmoothDampAngle(
                transform.eulerAngles.y,
                targetAngle,
                ref rotationVelocity,
                rotationSmoothTime
            );

            transform.rotation = Quaternion.Euler(0f, angle, 0f);
            moveDirection = Quaternion.Euler(0f, targetAngle, 0f) * Vector3.forward;
        }

        // Jump logic
        if (jumpPressed && isGrounded)
        {
            verticalVelocity = Mathf.Sqrt(jumpHeight * -2f * gravity);
        }

        // Apply gravity over time
        verticalVelocity += gravity * Time.deltaTime;

        // Combine horizontal movement and vertical velocity
        Vector3 finalVelocity = (moveDirection * moveSpeed) + (Vector3.up * verticalVelocity);
        controller.Move(finalVelocity * Time.deltaTime);
    }
}
```

---

## 3. Common Errors & Fixes Encountered

### 1. `CS1002: ; expected`
* **Cause:** Missing semicolon `;` or open bracket/parenthesis at or preceding the reported line.
* **Fix:** Ensure statements end with `;` and parentheses/brackets are properly closed.

### 2. `CS1061: 'Transform' does not contain a definition for 'main'`
* **Cause:** Calling `.main` on a `Transform` reference (like `cameraTransform.main`) instead of Unity's `Camera.main`.
* **Fix:** Use `Camera.main.transform` when retrieving the main camera reference.

### 3. `CS0103: The name 'cameraTranform' does not exist in the current context`
* **Cause:** Typo in variable name (missing the `s`).
* **Fix:** Spell it as `cameraTransform` matching the declaration at the top of the class.

### 4. `CS0103: The name 'mathf' does not exist in the current context`
* **Cause:** Lowercase `m` on `mathf`.
* **Fix:** Change `mathf` to `Mathf` (capital `M`).

### 5. `InvalidOperationException: You are trying to read Input using the UnityEngine.Input class...`
* **Cause:** Project Active Input Handling is set to Unity's **Input System Package (New)** while the script called legacy `UnityEngine.Input`.
* **Fix Option A:** Set **Edit > Project Settings > Player > Active Input Handling** to **Both**.
* **Fix Option B:** Use `UnityEngine.InputSystem` APIs (`Keyboard.current`) inside the script.

### 6. `CS0176: Member 'Keyboard.current' cannot be accessed with an instance reference` & `CS0165`
* **Cause:** Attempting to access `Keyboard.current` as an instance member on an unassigned variable named `keyboard.current`.
* **Fix:** Use `Keyboard.current` with a capital `K` to access the static property on the `Keyboard` type:
  ```csharp
  Keyboard keyboard = Keyboard.current;
  ```