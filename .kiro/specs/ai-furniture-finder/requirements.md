# Requirements Document

## Introduction

The AI Furniture Finder & Room Visualizer is a responsive web application that helps a user discover furniture that fits a physical space, matches style preferences, and stays within budget. The user uploads a photo of an empty area in a room, chooses a furniture category and placement area, provides room measurements anchored to at least one user-confirmed reference measurement, and sets search preferences. The System searches an authorized Amazon commerce integration, normalizes and ranks products against verified specifications, and generates approximate AI visualizations of eligible products placed in the user's room.

These requirements are derived from the approved design document (#[[file:design.md]]). The design prioritizes trustworthiness over convincing appearance: fit assessment is driven exclusively by verified product dimensions and user-confirmed measurements and is never influenced by the AI-generated image. Terminology in this document aligns with the design's data models and value types (FitStatus, ProviderMode, specificationCompleteness, measurementConfidence) and with the design's correctness properties P1 through P8.

## Glossary

- **System**: The AI Furniture Finder & Room Visualizer application as a whole, including client, server routes, and core services described in #[[file:design.md]].
- **Upload_Service**: The server-side service that validates, strips metadata from, stores, and deletes room photos.
- **Room_Analysis_Service**: The service that invokes the Vision_Provider to detect room features and builds the category-specific measurement form.
- **Vision_Provider**: The provider-abstraction implementation that detects room features and image-quality defects.
- **Product_Search_Service**: The service that queries the Commerce_Provider and returns raw product candidates.
- **Commerce_Provider**: The provider-abstraction implementation for commerce search; operates in ProviderMode LIVE or DEMO.
- **Filter_Rank_Pipeline**: The service that normalizes, filters, assigns FitStatus, and ranks products.
- **Visualization_Orchestrator**: The service that enqueues, tracks, cancels, and retries per-product visualization jobs.
- **ImageGen_Provider**: The provider-abstraction implementation for AI image generation; operates in ProviderMode LIVE or DEMO.
- **Session_Service**: The service that persists sessions, searches, results, and favorites.
- **Provider_Registry**: The component that resolves concrete providers from configuration and exposes system capabilities and modes.
- **FitStatus**: The verified-fit classification of a result, with values `FITS` or `NEEDS_VERIFICATION`.
- **ProviderMode**: The operational mode of a provider subsystem, with values `LIVE` or `DEMO`.
- **specificationCompleteness**: A product's dimension-completeness classification, with values `complete`, `partial`, or `missing`.
- **measurementConfidence**: A room project's measurement classification, with values `confirmed`, `estimated`, or `mixed`.
- **Reference_Measurement**: A user-confirmed real-world measurement used to anchor room dimensions.
- **Placement_Area**: The chosen placement location, with values `against_wall`, `beside_window`, `corner`, or `center`.
- **Canonical_Unit**: Millimeters (mm), the internal unit to which all dimensions are normalized.
- **Signed_View_Url**: A short-lived, access-controlled URL used to serve a stored image; never a raw public URL.
- **DEMO_Data**: Clearly labeled sample products or sample visualizations returned when credentials are absent.
- **Eligible_Product**: A product that passed all strict filters in the Filter_Rank_Pipeline and is included in results.
- **Homepage**: The application's landing page presenting the product value proposition and an entry point into the workflow.
- **Config_Example**: The `.env.example` file enumerating required environment variables without real secret values.

## Requirements

### Requirement 1: Homepage and Responsive Premium Interface

**User Story:** As a prospective user, I want a polished landing page that explains the product and how it works, so that I understand the value and can start the workflow with confidence.

#### Acceptance Criteria

1. WHEN a user visits the Homepage, THE System SHALL display a hero section presenting the product value proposition and a control that starts the furniture-finding workflow.
2. THE System SHALL display a three-step summary of the workflow on the Homepage.
3. THE System SHALL display a sample visualization on the Homepage that is labeled as an example.
4. THE System SHALL display a how-it-works section and a footer on the Homepage.
5. THE System SHALL render the Homepage as a responsive layout that adapts to large, medium, and small viewports.
6. WHEN a user activates the workflow-start control on the Homepage, THE System SHALL navigate the user to the room photo upload step.

### Requirement 2: Room Photo Upload

**User Story:** As a user, I want to upload a photo of an empty area in my room, so that the System can analyze my space and visualize furniture in it.

#### Acceptance Criteria

1. WHEN a user submits a file with MIME type JPEG, PNG, or WebP, greater than 0 bytes, and no larger than 10 MB, THE Upload_Service SHALL accept the file for storage within 5 seconds of receiving the complete file.
2. IF a user submits a file whose MIME type is not JPEG, PNG, or WebP, THEN THE Upload_Service SHALL reject the file before storage, persist no object, and return a message identifying the unsupported type and the supported types.
3. IF a user submits a file whose size exceeds 10 MB, THEN THE Upload_Service SHALL reject the file before storage, persist no object, and return a message identifying the size violation and the 10 MB maximum.
4. IF a user submits a file of exactly 0 bytes, THEN THE Upload_Service SHALL reject the file before storage, persist no object, and return a message indicating that the file is empty.
5. WHERE drag-and-drop is used, WHEN a user drops an image file onto the upload area, THE System SHALL accept the dropped file as an upload.
6. WHERE a file picker is used, WHEN a user selects an image file through the file picker, THE System SHALL accept the selected file as an upload.
7. WHEN an image is accepted, THE System SHALL display a preview of the uploaded image within 2 seconds of acceptance.
8. WHILE a preview is displayed, THE System SHALL provide controls that crop, replace, and remove the uploaded image.
9. WHEN an image is accepted for storage, THE Upload_Service SHALL strip EXIF and GPS metadata before persistence.
10. WHEN an image is persisted, THE Upload_Service SHALL store the image in access-controlled object storage and return a storage reference rather than a raw public URL.
11. WHEN the client requests to view a stored image, THE Upload_Service SHALL provide a Signed_View_Url whose time-to-live does not exceed 300 seconds.
12. IF an uploaded file is corrupt or cannot be decoded, THEN THE Upload_Service SHALL reject the upload, persist no partial object, and return a message stating the file could not be read and prompting the user to try a different file.

### Requirement 3: Furniture Category and Placement Selection

**User Story:** As a user, I want to choose a furniture category and placement area, so that the System tailors analysis, measurements, and search to what I am shopping for.

#### Acceptance Criteria

1. THE System SHALL present furniture categories including sofas, armchairs, coffee tables, dining tables, dining chairs, beds, desks, bookshelves, TV stands, wardrobes, nightstands, and a custom or other category.
2. WHEN a user selects a furniture category, THE System SHALL record the selected category for the room project.
3. THE System SHALL present placement areas `against_wall`, `beside_window`, `corner`, and `center`.
4. WHEN a user selects a placement area, THE System SHALL record the selected Placement_Area for the room project.
5. WHERE the custom or other category is selected, THE System SHALL accept a user-provided category label for the room project.
6. IF the System fails to record a selected furniture category, THEN THE System SHALL leave the room project without a valid furniture category and SHALL not retain any previously recorded category until recording succeeds.

### Requirement 4: AI Room Analysis

**User Story:** As a user, I want the System to analyze my room photo, so that it can identify the space and warn me about photo-quality problems.

#### Acceptance Criteria

1. WHEN a room photo and furniture category are submitted for analysis, THE Room_Analysis_Service SHALL invoke the Vision_Provider to detect walls, floor, windows, doors, corners, and obstacles, and SHALL return a result or defect within 30 seconds of submission.
2. WHEN the Vision_Provider returns a result reporting no image-quality defect, THE Room_Analysis_Service SHALL return the detected room features and a candidate Placement_Area with values drawn from `against_wall`, `beside_window`, `corner`, or `center`.
3. IF the Vision_Provider reports that the photo is too dark, THEN THE Room_Analysis_Service SHALL return the dark-image defect together with photography guidance describing the corrective action, SHALL withhold a candidate Placement_Area, and SHALL present a control that allows the user to re-upload a replacement photo.
4. IF the Vision_Provider reports that the photo is blurry, THEN THE Room_Analysis_Service SHALL return the blurry-image defect together with photography guidance describing the corrective action, SHALL withhold a candidate Placement_Area, and SHALL present a control that allows the user to re-upload a replacement photo.
5. IF the Vision_Provider reports insufficient coverage of the space, THEN THE Room_Analysis_Service SHALL return the insufficient-coverage defect together with photography guidance describing the corrective action, SHALL withhold a candidate Placement_Area, and SHALL present a control that allows the user to re-upload a replacement photo.
6. IF the Vision_Provider flags faces or personal content in the photo, THEN THE System SHALL return a warning recommending a photo without people, SHALL present a control that allows the user to re-upload a replacement photo, and SHALL remove the flagged content from object storage within 60 seconds after analysis processing completes.
7. WHEN the Room_Analysis_Service returns one or more AI-derived dimension values, THE System SHALL label each AI-derived value as an estimate and SHALL set the corresponding measurementConfidence contribution to `estimated`.
8. IF the Vision_Provider call does not return a result within 30 seconds, fails, or returns no result, THEN THE Room_Analysis_Service SHALL return an error indicating that analysis could not be completed, SHALL withhold detected room features and a candidate Placement_Area, and SHALL present a control that allows the user to retry analysis up to 3 times or re-upload a replacement photo.

### Requirement 5: Category-Specific Measurement Forms with Reference Anchoring

**User Story:** As a user, I want a measurement form tailored to my furniture category with unit choices and clear diagrams, so that I can provide accurate room dimensions and trust the resulting fit assessment.

#### Acceptance Criteria

1. WHEN a furniture category is selected, THE Room_Analysis_Service SHALL build a measurement form containing exactly the dimension fields defined for that category, with each field labeled by the dimension it represents.
2. IF a furniture category is selected for which no dimension fields are defined, THEN THE Room_Analysis_Service SHALL not build a measurement form and SHALL return a message indicating that no measurements are available for that category.
3. WHILE a measurement form is displayed, THE System SHALL allow the user to select a measurement unit of millimeters, centimeters, meters, inches, or feet for entered values.
4. WHEN a user enters or changes a measurement value in a field, THE System SHALL validate that the value is a number greater than 0 and no greater than 100000 in the selected unit, with no more than two decimal places.
5. IF a user enters a measurement value that is non-numeric, less than or equal to 0, greater than 100000 in the selected unit, or has more than two decimal places, THEN THE System SHALL reject that value, SHALL return a validation message identifying the invalid field, and SHALL retain all other previously entered measurement values.
6. WHEN a user changes the selected measurement unit while one or more values are entered, THE System SHALL re-validate each entered value against the bounds of the newly selected unit and SHALL return a validation message identifying any field that fails validation.
7. WHILE a measurement form is displayed, THE System SHALL display a visual diagram indicating which dimension each field represents.
8. WHEN a user changes a previously entered measurement value and that value passes validation, THE System SHALL replace the stored value for that field with the new value.
9. WHILE a measurement form is displayed, THE System SHALL provide back navigation to prior steps of the workflow while retaining previously entered measurement values.
10. WHILE no user-confirmed Reference_Measurement exists for the room project, THE System SHALL block search submission and SHALL return a message indicating that a reference measurement is required before fit can be assessed.
11. WHEN at least one user-confirmed Reference_Measurement exists and every entered measurement value is user-confirmed, THE System SHALL set measurementConfidence to `confirmed`.
12. WHEN at least one user-confirmed Reference_Measurement exists and at least one entered measurement value is an AI-derived estimate, THE System SHALL set measurementConfidence to `mixed`.
13. IF no user-confirmed Reference_Measurement exists for the room project, THEN THE System SHALL not set measurementConfidence to `confirmed`.

### Requirement 6: Search Filters

**User Story:** As a user, I want to specify required and optional search filters with strictness controls, so that results match my space, budget, and taste.

#### Acceptance Criteria

1. THE System SHALL require the user to provide a furniture category and maximum dimensions before search submission.
2. THE System SHALL allow the user to provide optional minimum dimensions.
3. THE System SHALL allow the user to provide optional filters including minimum price, maximum price, colors, materials, styles, brands, minimum rating, minimum review count, and condition.
4. THE System SHALL allow the user to provide a free-text description of the user's ideal furniture.
5. WHEN a user sets both a minimum price and a maximum price, THE System SHALL validate that the minimum price is less than or equal to the maximum price.
6. IF a user sets a minimum price greater than the maximum price, THEN THE System SHALL reject the budget range and return a validation message.
7. THE System SHALL allow the user to mark each filter field as strict or as a preference.
8. IF a field is marked strict but is absent or malformed, THEN THE System SHALL reject the search submission and return a message identifying the field.

### Requirement 7: Authorized Commerce Search and Mode Resolution

**User Story:** As a user, I want product search to use an authorized Amazon integration and to clearly tell me when it is showing sample data, so that I can trust what I see.

#### Acceptance Criteria

1. WHEN commerce credentials are present and pass validation at server boot, THE Provider_Registry SHALL bind a Commerce_Provider with ProviderMode `LIVE` and SHALL report commerce ProviderMode `LIVE` in the capabilities payload.
2. IF commerce credentials are absent, or are present but fail validation at server boot, THEN THE Provider_Registry SHALL bind a Commerce_Provider with ProviderMode `DEMO`.
3. WHILE the Commerce_Provider operates in ProviderMode `DEMO`, THE System SHALL return DEMO_Data in which every product carries a sample-data indicator set to true and SHALL report commerce ProviderMode `DEMO` in the capabilities payload so the client surfaces a demonstration-mode banner.
4. WHILE the Commerce_Provider operates in ProviderMode `DEMO`, THE System SHALL not present DEMO_Data as a live Amazon search.
5. THE Product_Search_Service SHALL return only product fields that are sourced from the Commerce_Provider and SHALL not fabricate product titles, specifications, prices, ratings, or links.
6. THE Product_Search_Service SHALL apply the Commerce_Provider's published rate limit, image-use policy, attribution, and affiliate-disclosure requirements on every commerce request.
7. WHEN a user chooses to view a product sourced from Amazon under affiliate authorization, THE Product_Search_Service SHALL resolve an outbound affiliate URL accompanied by the required affiliate-disclosure metadata.
8. WHEN a user chooses to view a product sourced from Amazon without affiliate authorization, THE Product_Search_Service SHALL resolve the canonical product listing URL.
9. WHEN a user chooses to view a product sourced from DEMO_Data, THE Product_Search_Service SHALL return a link carrying a demo indicator set to true and SHALL not present it as a live purchase path.
10. IF a commerce request exceeds the provider's rate limit, does not return within 10 seconds, or the provider returns an error, THEN THE Product_Search_Service SHALL return a provider-sourced error indicating the request could not be completed and SHALL not fabricate substitute products to fill the result set.

### Requirement 8: Normalize, Filter, and Rank Pipeline

**User Story:** As a user, I want results that honestly reflect what fits my space and budget, so that I never rely on a false fit claim.

#### Acceptance Criteria

1. WHEN raw products are received, THE Filter_Rank_Pipeline SHALL convert each product's dimension value on every required axis to the Canonical_Unit using the exact conversion factor and SHALL normalize price to a single canonical currency, such that no converted value differs from its exact mathematical conversion by more than 0.5 mm for dimensions or 0.01 unit for price.
2. IF a product's dimension unit or price currency is absent or cannot be converted to the Canonical_Unit or the canonical currency, THEN THE Filter_Rank_Pipeline SHALL treat the affected axis as not present for completeness classification and SHALL NOT reject the product solely on the basis of the unconverted value.
3. WHEN the Filter_Rank_Pipeline compares dimensions or price, THE Filter_Rank_Pipeline SHALL use only Canonical_Unit and canonical-currency values.
4. WHEN a product has specificationCompleteness `complete` on all required axes and at least one required axis strictly exceeds the maximum dimensions in Canonical_Unit, THE Filter_Rank_Pipeline SHALL reject the product from results.
5. WHEN a product has specificationCompleteness `complete` on all required axes and at least one required axis is strictly below the provided minimum dimensions in Canonical_Unit, THE Filter_Rank_Pipeline SHALL reject the product from results.
6. WHEN processing products, THE Filter_Rank_Pipeline SHALL apply the mandatory budget filter and every strict preference filter and SHALL reject any product whose canonical price is strictly less than the minimum price or strictly greater than the maximum price, or that fails any strict filter.
7. IF a product is missing one or more required dimension axes, THEN THE Filter_Rank_Pipeline SHALL set specificationCompleteness to `missing` when no required axis is present and to `partial` otherwise, and SHALL assign FitStatus `NEEDS_VERIFICATION`.
8. THE Filter_Rank_Pipeline SHALL assign FitStatus `FITS` only when a product has specificationCompleteness `complete` on all required axes and all required dimensions are within the maximum dimensions in Canonical_Unit, and SHALL assign FitStatus `NEEDS_VERIFICATION` to every other product.
9. WHEN eligible products are determined, THE Filter_Rank_Pipeline SHALL rank them in non-increasing order of a weighted score over dimensional fit, style, material, color, price, and rating.
10. WHEN two or more eligible products have an equal weighted score, THE Filter_Rank_Pipeline SHALL order those products by their original input order using a stable sort that preserves input sequence, so that the same input set always produces the same output order.
11. WHEN ranking is complete, THE Filter_Rank_Pipeline SHALL return a set of Eligible_Products containing no duplicate product and no larger than the configured maximum result count, where the configured maximum result count is a positive integer no greater than 100 and defaults to 24 when not configured.

### Requirement 9: Asynchronous AI Visualization

**User Story:** As a user, I want each matching product shown placed in my actual room, so that I can picture how it looks without being misled about whether it fits.

#### Acceptance Criteria

1. WHEN a search produces eligible products, THE Visualization_Orchestrator SHALL enqueue exactly one isolated visualization job per Eligible_Product with an initial state of pending.
2. THE Visualization_Orchestrator SHALL represent each visualization job in exactly one of the states pending, generating, completed, failed, or canceled, and SHALL only transition a job monotonically toward a terminal state of completed, failed, or canceled.
3. WHEN a visualization job is dispatched, THE Visualization_Orchestrator SHALL transition the job from pending to generating, SHALL invoke the ImageGen_Provider to generate an in-room image from the room photo and the product reference image, and upon success SHALL transition the job to completed with the generated image reference, which is present only while the job is completed.
4. IF the ImageGen_Provider does not return a generated image within 60 seconds for a job, THEN THE Visualization_Orchestrator SHALL set that job to failed with a timeout failure reason and SHALL leave every other job's state unchanged.
5. IF a visualization job fails, THEN THE Visualization_Orchestrator SHALL set only that job to failed with a failure reason that remains observable while the job is failed, and SHALL leave every other job's state unchanged.
6. WHEN a user requests to cancel a visualization job that is pending or generating, THE Visualization_Orchestrator SHALL set that job to canceled and SHALL return a cancellation confirmation.
7. IF a user requests to cancel a visualization job that is already in a terminal state, THEN THE Visualization_Orchestrator SHALL leave that job's state unchanged and return an indication that the job is already terminal.
8. WHEN a user requests to retry a failed visualization job and fewer than 3 retry attempts have been made for that job, THE Visualization_Orchestrator SHALL transition that job from failed to pending and re-run it independently of other jobs.
9. IF a user requests to retry a failed visualization job that has already reached 3 retry attempts, THEN THE Visualization_Orchestrator SHALL leave that job failed and return an indication that the retry limit has been reached.
10. WHERE the ImageGen_Provider policy permits caching, THE Visualization_Orchestrator SHALL cache a generated image keyed by the room image reference and the product image reference.
11. THE System SHALL label every visualization as an approximate representation.
12. THE System SHALL derive FitStatus from verified specifications and user-confirmed measurements only, independently of and invariant under any visualization job state.

### Requirement 10: Results Gallery and Product Detail

**User Story:** As a user, I want a responsive gallery of product cards and a detailed view for each product, so that I can evaluate matches and open the listing.

#### Acceptance Criteria

1. THE System SHALL display results in a responsive grid that uses three columns on large viewports, two columns on medium viewports, and one or two columns on small viewports.
2. WHERE a product card is displayed, THE System SHALL show the in-room visualization, product name, brand, price, key dimensions, color, material, rating, FitStatus, and a View Product control.
3. WHEN a user selects a product card, THE System SHALL open a detail view showing the visualization, the original room image, product specifications, the fit assessment, and the purchase link.
4. THE System SHALL provide sort options for the results gallery.
5. WHEN a user edits search criteria from the results view and requests regeneration, THE System SHALL re-run search and visualization for the updated criteria without restarting the workflow.
6. WHEN a product has no available in-room visualization, THE System SHALL display the product card without an in-room image.

### Requirement 11: Product Comparison

**User Story:** As a user, I want to compare selected products side by side, so that I can decide between candidates.

#### Acceptance Criteria

1. WHEN a user selects two or more products to compare, THE System SHALL display a comparison across dimensions, price, material, color, rating, clearance, FitStatus, and visualizations.
2. IF a user attempts to start a comparison with fewer than two products, THEN THE System SHALL reject the comparison and prompt the user to select at least two products.

### Requirement 12: Save, Revisit, and Optional Accounts

**User Story:** As a user, I want to save and revisit results, and optionally keep an account, so that I can continue later without being forced to sign up.

#### Acceptance Criteria

1. WHEN a user saves a product during a session, THE Session_Service SHALL persist the favorite for that session.
2. WHEN a user revisits a saved search within a session, THE Session_Service SHALL return the previously computed search results.
3. THE System SHALL allow a user to complete the full product journey without creating an account.
4. WHERE an account is enabled, THE System SHALL associate saved photos, favorites, and projects with the authenticated user.
5. WHEN an authenticated user requests deletion of a saved photo or project, THE System SHALL delete the requested photo or project.
6. IF an authenticated user requests deletion of a photo or project that does not exist, THEN THE System SHALL return a not-found response informing the user that the requested item could not be located.

### Requirement 13: Honest Fit and Edge-Case Handling

**User Story:** As a user, I want the System to be honest about fit and to handle problem cases gracefully, so that I can trust its recommendations.

#### Acceptance Criteria

1. THE System SHALL distinguish FitStatus `FITS` from FitStatus `NEEDS_VERIFICATION` in every result presentation.
2. THE System SHALL assign FitStatus `NEEDS_VERIFICATION` to every product with incomplete required dimensions and SHALL never assign such a product FitStatus `FITS`.
3. IF the Filter_Rank_Pipeline returns no products that meet all mandatory conditions, THEN THE System SHALL explain which constraints eliminated candidates and SHALL offer to relax a specified constraint.
4. WHEN a user relaxes a specified constraint after an empty result set, THE System SHALL re-run search without restarting the workflow.
5. IF a product's reference image is unusable for visualization, THEN THE System SHALL set that visualization to failed with a reason and SHALL display the product card without the in-room image.
6. IF product price or availability changes between retrieval and display, THEN THE System SHALL present the latest provider-sourced values and SHALL not present stale fabricated values.

### Requirement 14: Security, Validation, and Responsive Delivery

**User Story:** As a user, I want my credentials and photos handled securely and the interface to work smoothly across modern browsers, so that my data is protected and the experience is reliable.

#### Acceptance Criteria

1. THE System SHALL keep all Amazon, AI, and storage credentials in server-side custody and SHALL never serialize a credential into any client-bound response.
2. THE System SHALL originate every provider call from the server rather than from the browser.
3. WHEN the client receives system capabilities, THE Provider_Registry SHALL report each subsystem's ProviderMode as `LIVE` or `DEMO`.
4. WHILE any subsystem operates in ProviderMode `DEMO`, THE System SHALL disclose demonstration mode for that subsystem in the client interface.
5. WHEN the System receives user input at any workflow step, THE System SHALL validate the input before processing.
6. WHILE an asynchronous operation is in progress, THE System SHALL display a loading or progress state to the user.
7. THE System SHALL render a responsive interface that functions on current versions of mainstream desktop and mobile browsers.

### Requirement 15: Demonstration Mode, Configuration, and Verified Delivery

**User Story:** As an operator, I want clearly labeled demonstration mode, a secret-free configuration template, and automated tests, so that I can run and trust the system before connecting live integrations.

#### Acceptance Criteria

1. WHILE any subsystem operates in ProviderMode `DEMO`, THE System SHALL label the demonstration mode for that subsystem in the client interface so the user can distinguish sample data from live data.
2. THE System SHALL provide a Config_Example file that enumerates every required environment variable and SHALL contain no real secret values.
3. THE System SHALL provide automated tests covering dimension filtering, budget filtering, unit conversion, outbound product-link resolution, and visualization failure scenarios.
4. WHILE a commerce or image-generation integration is not connected and tested, THE System SHALL not represent that integration as operational.
5. WHEN commerce or image-generation credentials are absent, THE System SHALL activate ProviderMode `DEMO` automatically without manual configuration.
