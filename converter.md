#!/usr/bin/env python3
import json
import argparse
import sys


def load_insomnia_collection(path):
    """Load Insomnia JSON export from disk."""
    with open(path, 'r', encoding='utf-8') as f:
        return json.load(f)


def build_id_map(resources):
    """Map resource _id to resource object."""
    return {res['_id']: res for res in resources}


def transform_request(ins_req):
    """Transform an Insomnia request into Postman request format."""
    pm_req = {
        "name": ins_req.get("name"),
        "request": {
            "method": ins_req.get("method"),
            "header": [],
            "body": {},
            "url": {}
        }
    }

    # Headers
    for hdr in ins_req.get("headers", []):
        pm_req["request"]["header"].append({
            "key": hdr.get("name"),
            "value": hdr.get("value")
        })

    # Body
    body = ins_req.get("body", {})
    text = body.get("text")
    if text is not None:
        pm_req["request"]["body"] = {"mode": "raw", "raw": text}
        mime = body.get("mimeType")
        if mime:
            pm_req["request"]["body"]["options"] = {"raw": {"language": mime.split("/")[-1]}}

    # URL
    url = ins_req.get("url")
    if isinstance(url, str):
        pm_req["request"]["url"] = {"raw": url}
    else:
        pm_req["request"]["url"] = url

    return pm_req


def convert(insomnia_data, schema_identifier):
    """Convert entire Insomnia collection JSON into Postman collection JSON."""
    resources = insomnia_data.get("resources", [])
    workspace = next((r for r in resources if r.get("_type") == "workspace"), None)
    if workspace is None:
        raise ValueError("No Insomnia workspace found")

    collection = {
        "info": {"name": workspace.get("name", "Converted Collection"), "schema": schema_identifier},
        "item": []
    }

    # Prepare request groups (folders)
    folder_map = {}
    for r in resources:
        if r.get("_type") == "request_group":
            folder_map[r["_id"]] = {"name": r.get("name"), "item": []}

    # Process individual requests
    for r in resources:
        if r.get("_type") == "request":
            pm_req = transform_request(r)
            parent_id = r.get("parentId")
            if parent_id in folder_map:
                folder_map[parent_id]["item"].append(pm_req)
            else:
                collection["item"].append(pm_req)

    # Append folders with their items
    for folder in folder_map.values():
        if folder["item"]:
            collection["item"].append({"name": folder["name"], "item": folder["item"]})

    return collection


def main():
    parser = argparse.ArgumentParser(
        description="Convert Insomnia collection to Postman collection offline"
    )
    parser.add_argument("input", help="Path to Insomnia JSON export")
    parser.add_argument("output", help="Path to save Postman JSON")
    parser.add_argument(
        "--schema",
        required=True,
        help="Local identifier for Postman schema (e.g., filename or schema string)"
    )
    args = parser.parse_args()

    try:
        insomnia_data = load_insomnia_collection(args.input)
        postman_collection = convert(insomnia_data, args.schema)
        with open(args.output, "w", encoding="utf-8") as f:
            json.dump(postman_collection, f, indent=2)
        print(f"Postman collection saved to {args.output}")
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
