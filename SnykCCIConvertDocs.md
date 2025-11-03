# Convert project-scoped ignores to asset-scoped ignores

## [Docs](https://docs.snyk.io/manage-risk/prioritize-issues-for-fixing/ignore-issues/consistent-ignores-for-snyk-code/convert-project-scoped-ignores-to-asset-scoped-ignores#convert-ignores-using-snyk-api)

## Conversion setup

If you are new to Snyk Code Projects, you can skip this step — there are no existing ignores to convert.

Conversion lets you control which legacy ignores are converted and what metadata becomes the single source of truth. If you enabled Snyk Code Consistent Ignores and haven’t rescanned, you may need to retest the Project for the “Ignore across repository” button to become active.

## Convert an individual ignore

1. In the Snyk Web UI, open a Project and an issue card for an issue ignored before the feature was enabled.  
2. A warning marks ignores created by the legacy system. To convert the ignore to asset-scoped, click **Ignore across repository**.

### What metadata changes

- Ignored by and Timestamp: reflect the conversion time and user (not the original creator/time).  
- Reason, Description, and Expiration date: retained.

## Bulk ignore conversion

### Web UI (Projects with ≤ 200 legacy ignores)

Use the Projects page:

- Organization > Projects > click **Convert (number of) ignores in this Project**.

### Use Snyk API (Projects with > 200 ignores or custom control)

For large Projects or when you need precise control over ignore attributes, use the API to programmatically convert ignores.

## API-based conversion — high-level steps

1. List relevant Snyk Code Projects.  
2. Retrieve existing project-scoped ignores for a Project.  
3. Map project-scoped findings to asset-scoped findings.  
4. Create equivalent asset-scoped ignore policy.  
5. Delete the original project-scoped ignore.  
6. Verify the changes by retesting the Project.

---

## Step details

### 1) List relevant Snyk Code Projects

Goal: find the project_id for projects to convert.

Endpoint:

```
GET /rest/orgs/{org_id}/projects?types=sast
```

Response includes project_id, origin, and repo/branch info.

### 2) Retrieve existing project-scoped ignores

Goal: get all project-scoped ignores for a project.

Endpoint:

```curl

GET /v1/org/{org_id}/project/{project_id}/ignores

```

Response format (example):

```json

{
    "1ddad474-39f1-4ac4-b9c6-f2f79a65fd88": [
        {
            "reason": "The reason for the ignore",
            "created": "2025-04-03T12:19:23.852Z",
            "ignoredBy": {
                "id": "a1fd39ee-8253-4dab-8df9-8b5ba5bdcbc9",
                "name": "John Doe",
                "email": "john.doe@snyk.io"
            },
            "reasonType": "not-vulnerable"
        }
    ]
}
```

Note the project-scoped finding ID (key), reason, and reasonType for each ignore to convert.

### 3) Map project-scoped findings → asset-scoped findings

Goal: find the asset-scoped finding ID (snyk/asset/finding/v1) for each project-scoped ID.

Endpoint:

```curl
GET /rest/orgs/{org_id}/issues?scan_item.type=project&scan_item.id={project_id}
```

(Optionally add `&status=open`.)

Response excerpt (example):

```json

{
    "data": [
        {
            "key_asset": "33235ab8-2535-4f12-9115-2d57f41c81f1",
            "key": "1ddad474-39f1-4ac4-b9c6-f2f79a65fd88"
        }
    ]
}
```

Match each project-scoped key to its key_asset.

### 4) Create the asset-scoped ignore policy

Goal: create a policy that mirrors the original ignore but targets the asset-scoped finding.

Endpoint:

POST /rest/orgs/{org_id}/policies
Optionally, use &status=open if you only need to map open findings. Note that it is possible that ignores findings can also exist for resolved findings.

Response format: The response contains a list of issues within the Project.

```json

{
  "data": [
    {
    	    // ... other issue details
    	    "key_asset": "33235ab8-2535-4f12-9115-2d57f41c81f1", // <-- Asset-scoped ID
    	    "key": "1ddad474-39f1-4ac4-b9c6-f2f79a65fd88",   	// <-- Project-scoped ID
    	    // ... other issue details
	}
  ]
  // ... potentially more issues
}
```

For each Project-scoped finding ID (key) from step 2, find the matching issue in this response and extract its corresponding asset-scoped finding ID (key_asset). You will need this key_asset value to create the new ignore policy.

Create the new asset-scoped ignore policy
Goal: Create a new policy that replicates the original ignore but targets the asset-scoped finding ID obtained in step 3.

API Endpoint: POST /rest/orgs/{org_id}/policies

Payload: Construct the payload using the information gathered. Map the reasonType from step 2 to the ignore_type field, use the reason from step 2, and use the asset-scoped ID (key_asset) from step 3 as the value in the condition.

```json

{
  "data": {
	"attributes": {
  	"action": {
    	"data": {
      		"ignore_type": "not-vulnerable", // Mapped from reasonType in Step 2
      		"reason": "The reason for the ignore" // From reason in Step 2
    	}
  	},
  	"action_type": "ignore",
  	"conditions_group": {
    	"conditions": [
      	{
        	"field": "snyk/asset/finding/v1",
        	"operator": "includes",
        	"value": "33235ab8-2535-4f12-9115-2d57f41c81f1" // The key_asset value from Step 3
      	}
    	],
    		"logical_operator": "and" // Typically 'and' for single condition
  	},
  	"name": "Consistent Ignore - Converted [Optional Identifier]" // Choose a meaningful name
	},
	"type": "policy"
  }
}
```

Delete the original Project-scoped ignore
Goal: Remove the legacy Project-scoped ignore now that an equivalent asset-scoped policy exists.

API Endpoint: DELETE /v1/org/{org_id}/project/{project_id}/ignore/{project_scoped_id}

Key Information: Use the project_id from step 1 and the {project_scoped_id} which is the key from step 2 response (e.g., 1ddad474-39f1-4ac4-b9c6-f2f79a65fd88).

Verify the changes
Goal: Ensure the conversion was successful and the finding remains ignored under the new policy.

Action: Retest the relevant Project in Snyk (snyk code test or using the Snyk Web UI).

Check: Confirm that the finding previously covered by the Project-scoped ignore is still ignored.