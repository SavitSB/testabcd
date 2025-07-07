#!/usr/bin/env python3
"""
Convert an Insomnia collection JSON into a Postman collection JSON (v2.1.0).
Requires Python 3.
"""
import json
import argparse
import sys
from urllib.parse import urlparse, parse_qsl


def load_insomnia_collection(path):
    """Load Insomnia JSON export from disk."""
    with open(path, 'r', encoding='utf-8') as f:
        return json.load(f)


def build_item_from_request(ins_req):
    """Build a Postman request item following v2.1.0 schema."""
    name = ins_req.get('name', '')
    description = ins_req.get('description', '')

    # Construct the request object
    request = {
        'method': ins_req.get('method', 'GET'),
        'header': [],
        'body': {},
        'url': {}
    }

    # Headers
    for hdr in ins_req.get('headers', []):
        h = {'key': hdr.get('name', ''), 'value': hdr.get('value', '')}
        if 'description' in hdr:
            h['description'] = hdr['description']
        request['header'].append(h)

    # Body (raw text)
    body = ins_req.get('body', {})
    if body.get('text') is not None:
        request['body'] = {
            'mode': 'raw',
            'raw': body.get('text', '')
        }
        mime = body.get('mimeType')
        if mime:
            language = mime.split('/')[-1]
            request['body']['options'] = {'raw': {'language': language}}

    # URL: extract raw string or object field
    url_field = ins_req.get('url') or ''
    raw_url = url_field.get('raw') if isinstance(url_field, dict) else url_field
    parsed = urlparse(raw_url)

    # Build structured URL object
    url_obj = {'raw': raw_url}
    if parsed.scheme:
        url_obj['protocol'] = parsed.scheme
    if parsed.hostname:
        url_obj['host'] = parsed.hostname.split('.')
    path_segments = [seg for seg in parsed.path.split('/') if seg]
    if path_segments:
        url_obj['path'] = path_segments
    query_params = parse_qsl(parsed.query)
    if query_params:
        url_obj['query'] = [{'key': k, 'value': v} for k, v in query_params]
    if parsed.fragment:
        url_obj['hash'] = parsed.fragment

    request['url'] = url_obj

    # Assemble the item
    item = {
        'name': name,
        'request': request,
        'response': []
    }
    if description:
        item['description'] = description
    return item


def convert(insomnia_data, schema_identifier):
    """Convert Insomnia export into a Postman v2.1.0 collection."""
    resources = insomnia_data.get('resources', [])

    # Locate workspace for collection metadata
    workspace = next((r for r in resources if r.get('_type') == 'workspace'), None)
    if not workspace:
        raise ValueError('No workspace found in Insomnia export.')

    # Initialize collection
    collection = {
        'info': {
            'name': workspace.get('name', 'Converted Collection'),
            'schema': schema_identifier
        },
        'item': []
    }

    # Map request groups to folder containers
    folder_map = {
        r['_id']: {'name': r.get('name', ''), 'item': []}
        for r in resources if r.get('_type') == 'request_group'
    }

    # Process each request
    for res in resources:
        if res.get('_type') == 'request':
            item = build_item_from_request(res)
            parent_id = res.get('parentId')
            if parent_id in folder_map:
                folder_map[parent_id]['item'].append(item)
            else:
                collection['item'].append(item)

    # Attach folders with their items
    for folder in folder_map.values():
        if folder['item']:
            collection['item'].append({
                'name': folder['name'],
                'item': folder['item']
            })

    return collection


def main():
    parser = argparse.ArgumentParser(
        description='Convert Insomnia collection to Postman v2.1.0'
    )
    parser.add_argument('input', help='Path to Insomnia JSON export')
    parser.add_argument('output', help='Path to save Postman JSON')
    parser.add_argument(
        '--schema', required=True,
        help='Local schema identifier (e.g., path or schema string)'
    )
    args = parser.parse_args()

    try:
        insomnia_data = load_insomnia_collection(args.input)
        postman_collection = convert(insomnia_data, args.schema)
        with open(args.output, 'w', encoding='utf-8') as f:
            json.dump(postman_collection, f, indent=2)
        print(f'Postman v2.1.0 collection saved to {args.output}')
    except Exception as e:
        print(f'Error: {e}', file=sys.stderr)
        sys.exit(1)

if __name__ == '__main__':
    main()
