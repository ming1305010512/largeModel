### Code examples

```curl
curl -X POST 'https://api.firecrawl.dev/v2/scrape' \
-H 'Authorization: Bearer fc-c448154c745e4fc9b8e8d62af6336e42' \
-H 'Content-Type: application/json' \
-d $'{
  "url": "firecrawl.dev"
}'
```

```python
# pip install firecrawl-py
from firecrawl import Firecrawl

app = Firecrawl(api_key="fc-c448154c745e4fc9b8e8d62af6336e42")

# Scrape a website:
app.scrape('firecrawl.dev')
```

