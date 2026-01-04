!pip install pandas google-api-python-client
import pandas as pd
import re
from googleapiclient.discovery import build

# YouTube API anahtarınızı buraya yazın
API_KEY = "AIzaSyBg9d6ITXVi3SAn-h4ck3r"
youtube = build('youtube', 'v3', developerKey=API_KEY)

# Video ID'sini URL'den çıkarmak için fonksiyon
def extract_video_id(url):
    # Farklı YouTube URL formatlarını destekler
    patterns = [
        r'(?:v=|\/)([0-9A-Za-z_-]{11}).*',  # standart & youtu.be
        r'(?:youtu\.be\/)([0-9A-Za-z_-]{11})',
        r'(?:embed\/)([0-9A-Za-z_-]{11})'
    ]
    for pattern in patterns:
        match = re.search(pattern, url)
        if match:
            return match.group(1)
    return None

# CSV dosyasını oku
df = pd.read_csv('videolar.csv')

# URL sütununda olmayan satırları temizle
df = df.dropna(subset=['url'])

# Video ID'lerini çıkar
df['video_id'] = df['url'].apply(extract_video_id)

# Geçersiz ID'leri çıkar
df = df[df['video_id'].notna()]

# Benzersiz ID'leri al (API kotasını korumak için)
video_ids = df['video_id'].unique().tolist()

# En fazla 50 ID'yi bir seferde sorgulayabildiğimiz için parçalara böl
def chunk_list(lst, n):
    for i in range(0, len(lst), n):
        yield lst[i:i + n]

all_titles = {}

for chunk in chunk_list(video_ids, 50):  # YouTube API maksimum 50 ID/istek
    id_string = ','.join(chunk)
    request = youtube.videos().list(
        part='snippet',
        id=id_string
    )
    response = request.execute()
    
    for item in response['items']:
        all_titles[item['id']] = item['snippet']['title']

# Başlıkları DataFrame'e ekle
df['title'] = df['video_id'].map(all_titles)

# Sonucu yeni bir CSV'ye kaydet
df.to_csv('videolar_basliklarla.csv', index=False, encoding='utf-8')

print("Başlıklar başarıyla alındı ve 'videolar_basliklarla.csv' dosyasına kaydedildi.")
