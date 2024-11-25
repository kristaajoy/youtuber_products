# youtuber_products

script to fetch youtube videos -- 
```
import requests
import sqlite3

# Your YouTube API key here
API_KEY = 'youtube_API_key'

# Function to fetch YouTube videos based on a channel ID or keyword
def fetch_youtube_videos(channel_id, max_results=100):
    base_url = f'https://www.googleapis.com/youtube/v3/search?key={API_KEY}&channelId={channel_id}&part=snippet&type=video&maxResults={max_results}'
    
    videos = []
    next_page_token = None

    while True:
        # Add nextPageToken if it exists to get the next set of results
        url = base_url
        if next_page_token:
            url += f"&pageToken={next_page_token}"

        response = requests.get(url)
        data = response.json()

        # Debugging: Print the full response to see its structure
        print("API Response:", data)  # This will show the full response

        # Check if 'items' exists in the response, handle errors if it's missing
        if 'items' in data:
            videos.extend(data['items'])
        else:
            print("Error: 'items' not found in the API response.")
            # Optionally print the whole response to debug why 'items' is missing
            print("Full API Response:", data)

        # Check if there are more results and update nextPageToken
        next_page_token = data.get('nextPageToken')

        # If there are no more pages, break the loop
        if not next_page_token:
            break

    return videos

# Example: Fetch data for a channel
channel_id = 'CHANNEL_id'  # Replace with Channel ID
videos = fetch_youtube_videos(channel_id, max_results=50)  # Adjust max_results as needed

# Store the fetched data into SQLite database
conn = sqlite3.connect('youtube_youtuber_products.db')
cursor = conn.cursor()

for video in videos:
    title = video['snippet']['title']
    description = video['snippet']['description']
    url = f"https://www.youtube.com/watch?v={video['id']['videoId']}"
    publish_date = video['snippet']['publishedAt']

    # For simplicity, we'll just use placeholder values for views, likes, and comments
    views = likes = comments = 0  # You can fetch these values from a detailed API request later

    cursor.execute('''
    INSERT INTO youtube_videos (title, description, url, views, likes, comments, publish_date)
    VALUES (?, ?, ?, ?, ?, ?, ?)
    ''', (title, description, url, views, likes, comments, publish_date))

conn.commit()
conn.close()

print("Data stored successfully!")
```


- product information is based on US availability
- this might skew data slightly for products/youtubers based in other countries, such as Grog! being australian 

Query of average views and average likes if a company name, url, or product url is found in the description: 
```
SELECT company_found,
        ROUND(AVG(avg_views),2) AS total_avg_views,
        ROUND(AVG(avg_likes),2) AS total_avg_likes
FROM
    (SELECT v.channel,
        CASE 
          WHEN v.description LIKE '%'|| c.company ||'%' 
         OR v.description LIKE '%' || c.url || '%' 
         OR v.description LIKE '%' || p.product_url || '%'
            THEN 'Yes'
        WHEN v.description LIKE '%Code%' AND v.description NOT LIKE '%'|| c.company|| '%' THEN 'No'
        ELSE 'No'
    END AS company_found,
    COUNT( DISTINCT v.title) AS video_count, 
    ROUND(AVG(DISTINCT v.views), 2) AS avg_views,
    ROUND(AVG(DISTINCT v.likes), 2) AS avg_likes
FROM youtube_videos AS v
LEFT JOIN companies AS c
    ON c.channel = v.channel
LEFT JOIN youtuber_products AS p
    ON p.company = c.company
GROUP BY v.channel, company_found) AS company_views

GROUP BY company_found
ORDER BY total_avg_views DESC;
```
