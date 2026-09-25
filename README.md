# Helix Parser

Helix Parser reads a folder of documents in different formats (.txt, .md, .csv), 
extracts their content and metadata, and builds a searchable index of the knowledge base, saved as JSON.

## Requirements
- Python 3.11 or newer
- No external dependecies (standard library only)

## Installation
* Clone the repository and move into it
* Create and activate a virtual enviroments
* Install the package in editable mode:
```bash 
    pip install -e.
```
* Check that installation works:
```bash 
    python -c "import helix_parser: print('OK')"
```
