import json
import argparse
import sys

POSTMAN_SCHEMA = "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"


def load_insomnia_collection(path):
    with open(path, 'r', encoding='utf-8') as f:
        return json.load(f)


def build_id_map(resources):
    return {res['_id']: res for res in resources}


def transform_request(ins_req):
    # Basic transformation of an Insomnia request to Postman format
    request = {
        'name': ins_req.get('name'),
        'request': {
            'method': ins_req.get('method'),
            'header': [],
            'body': {},
            'url': {}
        }
    }

    # Headers
    for hdr in ins_req.get('headers', []):
        request['request']['header'].append({
            'key': hdr.get('name'),
            'value': hdr.get('value')
        })

    # Body
    body = ins_req.get('body', {})
    mime = body.get('mimeType')
    text = body.get('text')
    if text is not None:
        request['request']['body'] = {
            'mode': 'raw',
            'raw': text
        }
        if mime:
            request['request']['body']['options'] = {
                'raw': {'language': mime.split('/')[-1]}
            }

    # URL
    url = ins_req.get('url')
    if isinstance(url, str):
        # Raw URL string
        request['request']['url'] = {'raw': url}
    else:
        # Structured URL
        request['request']['url'] = url

    return request


def convert(insomnia_data):
    resources = insomnia_data.get('resources', [])
    id_map = build_id_map(resources)

    # Get workspace
    workspace = next((r for r in resources if r['_type'] == 'workspace'), None)
    if not workspace:
        raise ValueError('No Insomnia workspace found')

    workspace_id = workspace['_id']
    collection = {
        'info': {
            'name': workspace.get('name', 'Converted Collection'),
            'schema': POSTMAN_SCHEMA
        },
        'item': []
    }

    # Organize groups and requests
    # Map group id to folder
    groups = [r for r in resources if r['_type'] == 'request_group']
    folder_map = {g['_id']: {'name': g.get('name'), 'items': []} for g in groups}

    # Process requests
    for res in resources:
        if res['_type'] == 'request':
            parent = res.get('parentId')
            pm_req = transform_request(res)
            if parent and parent in folder_map:
                folder_map[parent]['items'].append(pm_req)
            else:
                # Goes to root
                collection['item'].append(pm_req)

    # Append folders with items
    for folder in folder_map.values():
        if folder['items']:
            collection['item'].append({
                'name': folder['name'],
                'item': folder['items']
            })

    return collection


def main():
    parser = argparse.ArgumentParser(description='Convert Insomnia collection to Postman collection')
    parser.add_argument('input', help='Path to Insomnia JSON export')
    parser.add_argument('output', help='Path to save Postman JSON')
    args = parser.parse_args()

    try:
        insomnia_data = load_insomnia_collection(args.input)
        postman_collection = convert(insomnia_data)
        with open(args.output, 'w', encoding='utf-8') as f:
            json.dump(postman_collection, f, indent=2)
        print(f"Postman collection saved to {args.output}")
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == '__main__':
    main()
