# NeetPrep Android App Deep Links

## Introduction

Deep links in the NeetPrep Android app allow direct navigation to specific sections or actions within the app, such as viewing course details, accessing test information, or checking order summaries. This document lists the available deep links, their parameters, and usage examples.

## Deep Links

### 1. General Navigation

1. **Open Fragment:** `https://www.neetprep.com/openFragment`
   - **Description**: Opens a specific fragment within the app. The target fragment depends on the app's internal logic or additional parameters (not specified here).

### 2. Purchase and Orders

1. **Order Summary:** `https://www.neetprep.com/buyNow/orderSummary`
   - **Description**: Opens the order summary page for purchases.
   - **Note**: This feature is currently under development and not fully functional.

### 3. Course and Product Information

1. https://www.neetprep.com/newui/product/3521--Target-or-anything
2. https://www.neetprep.com/newui/productPage?courseId=3521
3. https://www.neetprep.com/newui/productPage/productPage?courseId=3521
4. https://www.neetprep.com/course_details?id=3521

### 4. Test Information

Test information deep links provide access to various test-related features, such as sample tests, DPP (Daily Practice Problems) tests, topic-specific tests, and test results. They share a common base URL with customizable parameters.

1. **Test Info:** `https://www.neetprep.com/testinfo`
   - **Parameters**:
     - `testId`: Unique identifier of the test.
     - `testName`: Name of the test.
     - `isDPPTest` (optional): Boolean indicating if it’s a DPP test.
     - `topicId` (optional): Topic identifier associated with the test.
     - `buttonType` (optional): Specifies the action or view (e.g., "VIEW_RESULTS").
   - **Examples**:
       - https://www.neetprep.com/testinfo?testId=VGVzdDozOTAzNTgy&testName=DPPTest&isDppTest=true

## Notes

- Ensure all required parameters are included for deep links to work correctly.
- The **Order Summary** deep link is not yet fully implemented.
- For test information links, the `buttonType` parameter can trigger specific actions (e.g., viewing results).
- The `http` protocol in the full test example might be a typo or intentional; verify the correct protocol before use.

### 5. Settings

1. https://www.neetprep.com/android-settings: To open Android system settings for the app
2. https://www.neetprep.com/android-settings/notifications: To open notification settings for the app. Optional `channel` query param, that can take one of the following values. Example, `https://www.neetprep.com/android-settings/notifications?channel=com.lernr.app.PERSONALIZED`
   1. com.lernr.app.PERSONALIZED
   2. com.lernr.app.PROMOTIONAL
   3. com.lernr.app.MOTIVATIONAL
   4. com.lernr.app.GENERAL

## Changelog

Removed. See git history for changelog.
