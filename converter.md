#!/usr/bin/env python3
"""
Convert Insomnia collection JSON to a Postman collection JSON.
Requires Python 3.
"""
import json
import argparse
import sys
from urllib.parse import urlencode


def load_insomnia_collection(path):
    """Load Insomnia JSON export from disk."""
    with open(path, 'r', encoding='utf-8') as f:
        return json.load(f)


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
        pm_req["request"]["header"].append({"key": hdr.get("name"), "value": hdr.get("value")})

    # Body
    body = ins_req.get("body", {})
    text = body.get("text")
    if text is not None:
        pm_req["request"]["body"] = {"mode": "raw", "raw": text}
        mime = body.get("mimeType")
        if mime:
            pm_req["request"]["body"].setdefault("options", {})
            pm_req["request"]["body"]["options"]["raw"] = {"language": mime.split("/")[-1]}

    # URL reconstruction
    url_field = ins_req.get("url")
    raw_url = ""
    if isinstance(url_field, str):
        raw_url = url_field
    elif isinstance(url_field, dict):
        protocol = url_field.get("protocol", "")
        host = url_field.get("host")
        host_str = ".".join(host) if isinstance(host, list) else (host or "")
        path = url_field.get("path")
        path_str = "/".join(path) if isinstance(path, list) else (path or "")
        raw_url = f"{protocol}://{host_str}"
        if path_str:
            raw_url += f"/{path_str}"
        query = url_field.get("query", [])
        if query:
            params = {q.get("name"): q.get("value") for q in query}
            raw_url += f"?{urlencode(params)}"
    pm_req["request"]["url"] = {"raw": raw_url}

    return pm_req


def convert(insomnia_data, schema_identifier):
    """Convert Insomnia collection JSON into Postman collection JSON."""
    resources = insomnia_data.get("resources", [])
    workspace = next((r for r in resources if r.get("_type") == "workspace"), None)
    if workspace is None:
        raise ValueError("No Insomnia workspace found")

    collection = {
        "info": {"name": workspace.get("name", "Converted Collection"), "schema": schema_identifier},
        "item": []
    }

    # Prepare request groups as folders
    folder_map = {r["_id"]: {"name": r.get("name"), "item": []} for r in resources if r.get("_type") == "request_group"}

    # Process each request
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
        description="Convert Insomnia collection to Postman collection (Python 3 only)"
    )
    parser.add_argument("input", help="Path to Insomnia JSON export")
    parser.add_argument("output", help="Path to save Postman JSON")
    parser.add_argument(
        "--schema", required=True,
        help="Local schema identifier (e.g., filename or schema string)"
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
