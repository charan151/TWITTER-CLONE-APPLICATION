# TWITTER-CLONE-APPLICATION
# THIS IS THE CLONE OF X
# twitter_clone.py
import sqlite3
import hashlib
import os
import json
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
import re

class Database:
    def __init__(self, db_name='twitter_clone.db'):
        self.db_name = db_name
        self.init_database()
    
    def get_connection(self):
        conn = sqlite3.connect(self.db_name)
        conn.row_factory = sqlite3.Row
        return conn
    
    def init_database(self):
        conn = self.get_connection()
        cursor = conn.cursor()
        
        # Users table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                email TEXT UNIQUE NOT NULL,
                password TEXT NOT NULL,
                full_name TEXT,
                bio TEXT,
                location TEXT,
                website TEXT,
                profile_image TEXT,
                cover_image TEXT,
                verified INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        # Tweets table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS tweets (
                tweet_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                content TEXT NOT NULL,
                image_url TEXT,
                video_url TEXT,
                poll_id INTEGER,
                reply_to INTEGER,
                quote_tweet_id INTEGER,
                is_deleted INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                FOREIGN KEY (reply_to) REFERENCES tweets(tweet_id),
                FOREIGN KEY (quote_tweet_id) REFERENCES tweets(tweet_id)
            )
        ''')
        
        # Likes table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS likes (
                like_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                tweet_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id),
                UNIQUE(user_id, tweet_id)
            )
        ''')
        
        # Retweets table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS retweets (
                retweet_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                tweet_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id),
                UNIQUE(user_id, tweet_id)
            )
        ''')
        
        # Follows table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS follows (
                follow_id INTEGER PRIMARY KEY AUTOINCREMENT,
                follower_id INTEGER NOT NULL,
                following_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (follower_id) REFERENCES users(user_id),
                FOREIGN KEY (following_id) REFERENCES users(user_id),
                UNIQUE(follower_id, following_id)
            )
        ''')
        
        # Hashtags table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS hashtags (
                hashtag_id INTEGER PRIMARY KEY AUTOINCREMENT,
                tag TEXT UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        # Tweet_Hashtags table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS tweet_hashtags (
                tweet_id INTEGER NOT NULL,
                hashtag_id INTEGER NOT NULL,
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id),
                FOREIGN KEY (hashtag_id) REFERENCES hashtags(hashtag_id),
                PRIMARY KEY (tweet_id, hashtag_id)
            )
        ''')
        
        # Mentions table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS mentions (
                mention_id INTEGER PRIMARY KEY AUTOINCREMENT,
                tweet_id INTEGER NOT NULL,
                user_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id),
                FOREIGN KEY (user_id) REFERENCES users(user_id)
            )
        ''')
        
        # Direct Messages table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS direct_messages (
                message_id INTEGER PRIMARY KEY AUTOINCREMENT,
                sender_id INTEGER NOT NULL,
                recipient_id INTEGER NOT NULL,
                content TEXT NOT NULL,
                image_url TEXT,
                is_read INTEGER DEFAULT 0,
                is_deleted INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (sender_id) REFERENCES users(user_id),
                FOREIGN KEY (recipient_id) REFERENCES users(user_id)
            )
        ''')
        
        # Notifications table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS notifications (
                notification_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                type TEXT NOT NULL,
                content TEXT NOT NULL,
                related_user_id INTEGER,
                related_tweet_id INTEGER,
                is_read INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id)
            )
        ''')
        
        # Bookmarks table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS bookmarks (
                bookmark_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                tweet_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id),
                UNIQUE(user_id, tweet_id)
            )
        ''')
        
        # Lists table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS lists (
                list_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                name TEXT NOT NULL,
                description TEXT,
                is_private INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(user_id)
            )
        ''')
        
        # List Members table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS list_members (
                list_id INTEGER NOT NULL,
                user_id INTEGER NOT NULL,
                added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (list_id) REFERENCES lists(list_id),
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                PRIMARY KEY (list_id, user_id)
            )
        ''')
        
        # Polls table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS polls (
                poll_id INTEGER PRIMARY KEY AUTOINCREMENT,
                tweet_id INTEGER NOT NULL,
                duration_hours INTEGER DEFAULT 24,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                ends_at TIMESTAMP,
                FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id)
            )
        ''')
        
        # Poll Options table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS poll_options (
                option_id INTEGER PRIMARY KEY AUTOINCREMENT,
                poll_id INTEGER NOT NULL,
                option_text TEXT NOT NULL,
                votes INTEGER DEFAULT 0,
                FOREIGN KEY (poll_id) REFERENCES polls(poll_id)
            )
        ''')
        
        # Poll Votes table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS poll_votes (
                vote_id INTEGER PRIMARY KEY AUTOINCREMENT,
                poll_id INTEGER NOT NULL,
                option_id INTEGER NOT NULL,
                user_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (poll_id) REFERENCES polls(poll_id),
                FOREIGN KEY (option_id) REFERENCES poll_options(option_id),
                FOREIGN KEY (user_id) REFERENCES users(user_id),
                UNIQUE(poll_id, user_id)
            )
        ''')
        
        # Blocks table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS blocks (
                block_id INTEGER PRIMARY KEY AUTOINCREMENT,
                blocker_id INTEGER NOT NULL,
                blocked_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (blocker_id) REFERENCES users(user_id),
                FOREIGN KEY (blocked_id) REFERENCES users(user_id),
                UNIQUE(blocker_id, blocked_id)
            )
        ''')
        
        # Mutes table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS mutes (
                mute_id INTEGER PRIMARY KEY AUTOINCREMENT,
                muter_id INTEGER NOT NULL,
                muted_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (muter_id) REFERENCES users(user_id),
                FOREIGN KEY (muted_id) REFERENCES users(user_id),
                UNIQUE(muter_id, muted_id)
            )
        ''')
        
        # Trending topics view
        cursor.execute('''
            CREATE VIEW IF NOT EXISTS trending_hashtags AS
            SELECT h.tag, COUNT(th.tweet_id) as tweet_count
            FROM hashtags h
            JOIN tweet_hashtags th ON h.hashtag_id = th.hashtag_id
            JOIN tweets t ON th.tweet_id = t.tweet_id
            WHERE t.created_at >= datetime('now', '-24 hours')
            GROUP BY h.tag
            ORDER BY tweet_count DESC
            LIMIT 10
        ''')
        
        conn.commit()
        conn.close()

class TwitterClone:
    def __init__(self):
        self.db = Database()
        self.current_user = None
    
    @staticmethod
    def hash_password(password: str) -> str:
        """Hash password using SHA-256"""
        return hashlib.sha256(password.encode()).hexdigest()
    
    def register(self, username: str, email: str, password: str, full_name: str = "") -> Tuple[bool, str]:
        """Register a new user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        try:
            hashed_password = self.hash_password(password)
            cursor.execute('''
                INSERT INTO users (username, email, password, full_name)
                VALUES (?, ?, ?, ?)
            ''', (username, email, hashed_password, full_name))
            conn.commit()
            conn.close()
            return True, "Registration successful"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "Username or email already exists"
    
    def login(self, username: str, password: str) -> Tuple[bool, str]:
        """Login user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        hashed_password = self.hash_password(password)
        cursor.execute('''
            SELECT * FROM users WHERE username = ? AND password = ?
        ''', (username, hashed_password))
        
        user = cursor.fetchone()
        conn.close()
        
        if user:
            self.current_user = dict(user)
            return True, f"Welcome back, {username}!"
        return False, "Invalid credentials"
    
    def logout(self):
        """Logout current user"""
        self.current_user = None
    
    def update_profile(self, bio: str = None, location: str = None, 
                       website: str = None, full_name: str = None) -> Tuple[bool, str]:
        """Update user profile"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        updates = []
        params = []
        
        if bio is not None:
            updates.append("bio = ?")
            params.append(bio)
        if location is not None:
            updates.append("location = ?")
            params.append(location)
        if website is not None:
            updates.append("website = ?")
            params.append(website)
        if full_name is not None:
            updates.append("full_name = ?")
            params.append(full_name)
        
        if not updates:
            return False, "No updates provided"
        
        params.append(self.current_user['user_id'])
        query = f"UPDATE users SET {', '.join(updates)}, updated_at = CURRENT_TIMESTAMP WHERE user_id = ?"
        
        cursor.execute(query, params)
        conn.commit()
        conn.close()
        
        return True, "Profile updated successfully"
    
    def extract_hashtags(self, content: str) -> List[str]:
        """Extract hashtags from content"""
        return re.findall(r'#(\w+)', content)
    
    def extract_mentions(self, content: str) -> List[str]:
        """Extract mentions from content"""
        return re.findall(r'@(\w+)', content)
    
    def post_tweet(self, content: str, image_url: str = None, 
                   reply_to: int = None, quote_tweet_id: int = None) -> Tuple[bool, str, int]:
        """Post a new tweet"""
        if not self.current_user:
            return False, "Please login first", 0
        
        if len(content) > 280:
            return False, "Tweet exceeds 280 characters", 0
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        try:
            # Insert tweet
            cursor.execute('''
                INSERT INTO tweets (user_id, content, image_url, reply_to, quote_tweet_id)
                VALUES (?, ?, ?, ?, ?)
            ''', (self.current_user['user_id'], content, image_url, reply_to, quote_tweet_id))
            
            tweet_id = cursor.lastrowid
            
            # Extract and save hashtags
            hashtags = self.extract_hashtags(content)
            for tag in hashtags:
                cursor.execute('INSERT OR IGNORE INTO hashtags (tag) VALUES (?)', (tag.lower(),))
                cursor.execute('SELECT hashtag_id FROM hashtags WHERE tag = ?', (tag.lower(),))
                hashtag_id = cursor.fetchone()[0]
                cursor.execute('INSERT INTO tweet_hashtags VALUES (?, ?)', (tweet_id, hashtag_id))
            
            # Extract and save mentions
            mentions = self.extract_mentions(content)
            for mention in mentions:
                cursor.execute('SELECT user_id FROM users WHERE username = ?', (mention,))
                mentioned_user = cursor.fetchone()
                if mentioned_user:
                    cursor.execute('INSERT INTO mentions VALUES (NULL, ?, ?, CURRENT_TIMESTAMP)', 
                                 (tweet_id, mentioned_user[0]))
                    # Create notification
                    self.create_notification(mentioned_user[0], 'mention', 
                                           f'@{self.current_user["username"]} mentioned you',
                                           self.current_user['user_id'], tweet_id)
            
            # If it's a reply, notify the original tweet author
            if reply_to:
                cursor.execute('SELECT user_id FROM tweets WHERE tweet_id = ?', (reply_to,))
                original_author = cursor.fetchone()
                if original_author and original_author[0] != self.current_user['user_id']:
                    self.create_notification(original_author[0], 'reply',
                                           f'@{self.current_user["username"]} replied to your tweet',
                                           self.current_user['user_id'], tweet_id)
            
            conn.commit()
            conn.close()
            return True, "Tweet posted successfully", tweet_id
        except Exception as e:
            conn.close()
            return False, str(e), 0
    
    def delete_tweet(self, tweet_id: int) -> Tuple[bool, str]:
        """Delete a tweet (soft delete)"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM tweets WHERE tweet_id = ?', (tweet_id,))
        tweet = cursor.fetchone()
        
        if not tweet:
            conn.close()
            return False, "Tweet not found"
        
        if tweet['user_id'] != self.current_user['user_id']:
            conn.close()
            return False, "You can only delete your own tweets"
        
        cursor.execute('UPDATE tweets SET is_deleted = 1 WHERE tweet_id = ?', (tweet_id,))
        conn.commit()
        conn.close()
        
        return True, "Tweet deleted"
    
    def like_tweet(self, tweet_id: int) -> Tuple[bool, str]:
        """Like/unlike a tweet"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        # Check if already liked
        cursor.execute('''
            SELECT like_id FROM likes WHERE user_id = ? AND tweet_id = ?
        ''', (self.current_user['user_id'], tweet_id))
        
        existing_like = cursor.fetchone()
        
        if existing_like:
            cursor.execute('DELETE FROM likes WHERE like_id = ?', (existing_like[0],))
            conn.commit()
            conn.close()
            return True, "Tweet unliked"
        else:
            cursor.execute('''
                INSERT INTO likes (user_id, tweet_id) VALUES (?, ?)
            ''', (self.current_user['user_id'], tweet_id))
            
            # Notify tweet author
            cursor.execute('SELECT user_id FROM tweets WHERE tweet_id = ?', (tweet_id,))
            tweet_author = cursor.fetchone()
            if tweet_author and tweet_author[0] != self.current_user['user_id']:
                self.create_notification(tweet_author[0], 'like',
                                       f'@{self.current_user["username"]} liked your tweet',
                                       self.current_user['user_id'], tweet_id)
            
            conn.commit()
            conn.close()
            return True, "Tweet liked"
    
    def retweet(self, tweet_id: int) -> Tuple[bool, str]:
        """Retweet/unretweet a tweet"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT retweet_id FROM retweets WHERE user_id = ? AND tweet_id = ?
        ''', (self.current_user['user_id'], tweet_id))
        
        existing_retweet = cursor.fetchone()
        
        if existing_retweet:
            cursor.execute('DELETE FROM retweets WHERE retweet_id = ?', (existing_retweet[0],))
            conn.commit()
            conn.close()
            return True, "Retweet removed"
        else:
            cursor.execute('''
                INSERT INTO retweets (user_id, tweet_id) VALUES (?, ?)
            ''', (self.current_user['user_id'], tweet_id))
            
            # Notify tweet author
            cursor.execute('SELECT user_id FROM tweets WHERE tweet_id = ?', (tweet_id,))
            tweet_author = cursor.fetchone()
            if tweet_author and tweet_author[0] != self.current_user['user_id']:
                self.create_notification(tweet_author[0], 'retweet',
                                       f'@{self.current_user["username"]} retweeted your tweet',
                                       self.current_user['user_id'], tweet_id)
            
            conn.commit()
            conn.close()
            return True, "Retweeted"
    
    def follow_user(self, username: str) -> Tuple[bool, str]:
        """Follow a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        if target_user['user_id'] == self.current_user['user_id']:
            conn.close()
            return False, "You cannot follow yourself"
        
        try:
            cursor.execute('''
                INSERT INTO follows (follower_id, following_id) VALUES (?, ?)
            ''', (self.current_user['user_id'], target_user['user_id']))
            
            # Create notification
            self.create_notification(target_user['user_id'], 'follow',
                                   f'@{self.current_user["username"]} started following you',
                                   self.current_user['user_id'])
            
            conn.commit()
            conn.close()
            return True, f"Now following {username}"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "Already following this user"
    
    def unfollow_user(self, username: str) -> Tuple[bool, str]:
        """Unfollow a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        cursor.execute('''
            DELETE FROM follows WHERE follower_id = ? AND following_id = ?
        ''', (self.current_user['user_id'], target_user['user_id']))
        
        conn.commit()
        conn.close()
        return True, f"Unfollowed {username}"
    
    def block_user(self, username: str) -> Tuple[bool, str]:
        """Block a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        try:
            # Remove follow relationships
            cursor.execute('DELETE FROM follows WHERE follower_id = ? AND following_id = ?',
                         (self.current_user['user_id'], target_user['user_id']))
            cursor.execute('DELETE FROM follows WHERE follower_id = ? AND following_id = ?',
                         (target_user['user_id'], self.current_user['user_id']))
            
            # Add block
            cursor.execute('INSERT INTO blocks (blocker_id, blocked_id) VALUES (?, ?)',
                         (self.current_user['user_id'], target_user['user_id']))
            
            conn.commit()
            conn.close()
            return True, f"Blocked {username}"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "Already blocked this user"
    
    def unblock_user(self, username: str) -> Tuple[bool, str]:
        """Unblock a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        cursor.execute('DELETE FROM blocks WHERE blocker_id = ? AND blocked_id = ?',
                     (self.current_user['user_id'], target_user['user_id']))
        
        conn.commit()
        conn.close()
        return True, f"Unblocked {username}"
    
    def mute_user(self, username: str) -> Tuple[bool, str]:
        """Mute a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        try:
            cursor.execute('INSERT INTO mutes (muter_id, muted_id) VALUES (?, ?)',
                         (self.current_user['user_id'], target_user['user_id']))
            conn.commit()
            conn.close()
            return True, f"Muted {username}"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "Already muted this user"
    
    def unmute_user(self, username: str) -> Tuple[bool, str]:
        """Unmute a user"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        cursor.execute('DELETE FROM mutes WHERE muter_id = ? AND muted_id = ?',
                     (self.current_user['user_id'], target_user['user_id']))
        
        conn.commit()
        conn.close()
        return True, f"Unmuted {username}"
    
    def bookmark_tweet(self, tweet_id: int) -> Tuple[bool, str]:
        """Bookmark/unbookmark a tweet"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT bookmark_id FROM bookmarks WHERE user_id = ? AND tweet_id = ?
        ''', (self.current_user['user_id'], tweet_id))
        
        existing_bookmark = cursor.fetchone()
        
        if existing_bookmark:
            cursor.execute('DELETE FROM bookmarks WHERE bookmark_id = ?', (existing_bookmark[0],))
            conn.commit()
            conn.close()
            return True, "Bookmark removed"
        else:
            cursor.execute('INSERT INTO bookmarks (user_id, tweet_id) VALUES (?, ?)',
                         (self.current_user['user_id'], tweet_id))
            conn.commit()
            conn.close()
            return True, "Tweet bookmarked"
    
    def get_timeline(self, limit: int = 50) -> List[Dict]:
        """Get timeline of tweets"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT DISTINCT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count,
                   EXISTS(SELECT 1 FROM likes WHERE tweet_id = t.tweet_id AND user_id = ?) as liked_by_me,
                   EXISTS(SELECT 1 FROM retweets WHERE tweet_id = t.tweet_id AND user_id = ?) as retweeted_by_me,
                   EXISTS(SELECT 1 FROM bookmarks WHERE tweet_id = t.tweet_id AND user_id = ?) as bookmarked_by_me
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            LEFT JOIN follows f ON t.user_id = f.following_id
            LEFT JOIN retweets r ON t.tweet_id = r.tweet_id
            WHERE (f.follower_id = ? OR t.user_id = ? OR r.user_id IN (
                SELECT following_id FROM follows WHERE follower_id = ?
            ))
            AND t.is_deleted = 0
            AND t.user_id NOT IN (SELECT blocked_id FROM blocks WHERE blocker_id = ?)
            AND t.user_id NOT IN (SELECT muted_id FROM mutes WHERE muter_id = ?)
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (self.current_user['user_id'], self.current_user['user_id'], 
              self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], self.current_user['user_id'], limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_user_tweets(self, username: str, limit: int = 50) -> List[Dict]:
        """Get tweets from a specific user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            WHERE u.username = ? AND t.is_deleted = 0 AND t.reply_to IS NULL
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_user_replies(self, username: str, limit: int = 50) -> List[Dict]:
        """Get replies from a specific user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            WHERE u.username = ? AND t.is_deleted = 0 AND t.reply_to IS NOT NULL
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_user_media(self, username: str, limit: int = 50) -> List[Dict]:
        """Get tweets with media from a specific user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            WHERE u.username = ? AND t.is_deleted = 0 
            AND (t.image_url IS NOT NULL OR t.video_url IS NOT NULL)
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_user_likes(self, username: str, limit: int = 50) -> List[Dict]:
        """Get tweets liked by a specific user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count,
                   l.created_at as liked_at
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            JOIN likes l ON t.tweet_id = l.tweet_id
            JOIN users lu ON l.user_id = lu.user_id
            WHERE lu.username = ? AND t.is_deleted = 0
            ORDER BY l.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_bookmarks(self, limit: int = 50) -> List[Dict]:
        """Get bookmarked tweets"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count,
                   b.created_at as bookmarked_at
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            JOIN bookmarks b ON t.tweet_id = b.tweet_id
            WHERE b.user_id = ? AND t.is_deleted = 0
            ORDER BY b.created_at DESC
            LIMIT ?
        ''', (self.current_user['user_id'], limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def search_tweets(self, query: str, limit: int = 50) -> List[Dict]:
        """Search tweets by content"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            WHERE t.content LIKE ? AND t.is_deleted = 0
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (f'%{query}%', limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def search_users(self, query: str, limit: int = 20) -> List[Dict]:
        """Search users by username or full name"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT u.*,
                   (SELECT COUNT(*) FROM follows WHERE following_id = u.user_id) as followers_count,
                   (SELECT COUNT(*) FROM follows WHERE follower_id = u.user_id) as following_count,
                   (SELECT COUNT(*) FROM tweets WHERE user_id = u.user_id AND is_deleted = 0) as tweet_count
            FROM users u
            WHERE u.username LIKE ? OR u.full_name LIKE ?
            LIMIT ?
        ''', (f'%{query}%', f'%{query}%', limit))
        
        users = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return users
    
    def get_hashtag_tweets(self, hashtag: str, limit: int = 50) -> List[Dict]:
        """Get tweets for a specific hashtag"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            JOIN tweet_hashtags th ON t.tweet_id = th.tweet_id
            JOIN hashtags h ON th.hashtag_id = h.hashtag_id
            WHERE h.tag = ? AND t.is_deleted = 0
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (hashtag.lower(), limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def get_trending_hashtags(self, limit: int = 10) -> List[Dict]:
        """Get trending hashtags"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM trending_hashtags LIMIT ?', (limit,))
        
        trending = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return trending
    
    def get_tweet_details(self, tweet_id: int) -> Optional[Dict]:
        """Get detailed information about a tweet"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            WHERE t.tweet_id = ? AND t.is_deleted = 0
        ''', (tweet_id,))
        
        tweet = cursor.fetchone()
        
        if tweet:
            tweet_dict = dict(tweet)
            
            # Get replies
            cursor.execute('''
                SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                       (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                       (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count
                FROM tweets t
                JOIN users u ON t.user_id = u.user_id
                WHERE t.reply_to = ? AND t.is_deleted = 0
                ORDER BY t.created_at ASC
            ''', (tweet_id,))
            
            tweet_dict['replies'] = [dict(row) for row in cursor.fetchall()]
            conn.close()
            return tweet_dict
        
        conn.close()
        return None
    
    def get_user_profile(self, username: str) -> Optional[Dict]:
        """Get user profile information"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT u.*,
                   (SELECT COUNT(*) FROM follows WHERE following_id = u.user_id) as followers_count,
                   (SELECT COUNT(*) FROM follows WHERE follower_id = u.user_id) as following_count,
                   (SELECT COUNT(*) FROM tweets WHERE user_id = u.user_id AND is_deleted = 0) as tweet_count
            FROM users u
            WHERE u.username = ?
        ''', (username,))
        
        user = cursor.fetchone()
        
        if user:
            user_dict = dict(user)
            
            # Check if current user follows this user
            if self.current_user:
                cursor.execute('''
                    SELECT 1 FROM follows 
                    WHERE follower_id = ? AND following_id = ?
                ''', (self.current_user['user_id'], user_dict['user_id']))
                user_dict['followed_by_me'] = cursor.fetchone() is not None
                
                cursor.execute('''
                    SELECT 1 FROM follows 
                    WHERE follower_id = ? AND following_id = ?
                ''', (user_dict['user_id'], self.current_user['user_id']))
                user_dict['follows_me'] = cursor.fetchone() is not None
                
                cursor.execute('''
                    SELECT 1 FROM blocks 
                    WHERE blocker_id = ? AND blocked_id = ?
                ''', (self.current_user['user_id'], user_dict['user_id']))
                user_dict['blocked_by_me'] = cursor.fetchone() is not None
                
                cursor.execute('''
                    SELECT 1 FROM mutes 
                    WHERE muter_id = ? AND muted_id = ?
                ''', (self.current_user['user_id'], user_dict['user_id']))
                user_dict['muted_by_me'] = cursor.fetchone() is not None
            
            conn.close()
            return user_dict
        
        conn.close()
        return None
    
    def get_followers(self, username: str, limit: int = 50) -> List[Dict]:
        """Get followers of a user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT u.*,
                   (SELECT COUNT(*) FROM follows WHERE following_id = u.user_id) as followers_count,
                   (SELECT COUNT(*) FROM follows WHERE follower_id = u.user_id) as following_count
            FROM users u
            JOIN follows f ON u.user_id = f.follower_id
            JOIN users target ON f.following_id = target.user_id
            WHERE target.username = ?
            ORDER BY f.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        followers = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return followers
    
    def get_following(self, username: str, limit: int = 50) -> List[Dict]:
        """Get users followed by a user"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT u.*,
                   (SELECT COUNT(*) FROM follows WHERE following_id = u.user_id) as followers_count,
                   (SELECT COUNT(*) FROM follows WHERE follower_id = u.user_id) as following_count
            FROM users u
            JOIN follows f ON u.user_id = f.following_id
            JOIN users target ON f.follower_id = target.user_id
            WHERE target.username = ?
            ORDER BY f.created_at DESC
            LIMIT ?
        ''', (username, limit))
        
        following = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return following
    
    def send_direct_message(self, recipient_username: str, content: str, 
                           image_url: str = None) -> Tuple[bool, str]:
        """Send a direct message"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (recipient_username,))
        recipient = cursor.fetchone()
        
        if not recipient:
            conn.close()
            return False, "User not found"
        
        # Check if blocked
        cursor.execute('''
            SELECT 1 FROM blocks 
            WHERE (blocker_id = ? AND blocked_id = ?) 
            OR (blocker_id = ? AND blocked_id = ?)
        ''', (self.current_user['user_id'], recipient['user_id'],
              recipient['user_id'], self.current_user['user_id']))
        
        if cursor.fetchone():
            conn.close()
            return False, "Cannot send message to this user"
        
        cursor.execute('''
            INSERT INTO direct_messages (sender_id, recipient_id, content, image_url)
            VALUES (?, ?, ?, ?)
        ''', (self.current_user['user_id'], recipient['user_id'], content, image_url))
        
        # Create notification
        self.create_notification(recipient['user_id'], 'message',
                               f'New message from @{self.current_user["username"]}',
                               self.current_user['user_id'])
        
        conn.commit()
        conn.close()
        return True, "Message sent"
    
    def get_direct_messages(self, other_username: str, limit: int = 50) -> List[Dict]:
        """Get direct message conversation with a user"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (other_username,))
        other_user = cursor.fetchone()
        
        if not other_user:
            conn.close()
            return []
        
        cursor.execute('''
            SELECT dm.*, 
                   sender.username as sender_username,
                   recipient.username as recipient_username
            FROM direct_messages dm
            JOIN users sender ON dm.sender_id = sender.user_id
            JOIN users recipient ON dm.recipient_id = recipient.user_id
            WHERE ((dm.sender_id = ? AND dm.recipient_id = ?)
            OR (dm.sender_id = ? AND dm.recipient_id = ?))
            AND dm.is_deleted = 0
            ORDER BY dm.created_at ASC
            LIMIT ?
        ''', (self.current_user['user_id'], other_user['user_id'],
              other_user['user_id'], self.current_user['user_id'], limit))
        
        messages = [dict(row) for row in cursor.fetchall()]
        
        # Mark messages as read
        cursor.execute('''
            UPDATE direct_messages
            SET is_read = 1
            WHERE recipient_id = ? AND sender_id = ? AND is_read = 0
        ''', (self.current_user['user_id'], other_user['user_id']))
        
        conn.commit()
        conn.close()
        return messages
    
    def get_message_conversations(self) -> List[Dict]:
        """Get list of message conversations"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT DISTINCT
                CASE 
                    WHEN dm.sender_id = ? THEN dm.recipient_id
                    ELSE dm.sender_id
                END as other_user_id,
                u.username, u.full_name, u.profile_image,
                (SELECT content FROM direct_messages 
                 WHERE (sender_id = ? AND recipient_id = other_user_id)
                 OR (sender_id = other_user_id AND recipient_id = ?)
                 ORDER BY created_at DESC LIMIT 1) as last_message,
                (SELECT created_at FROM direct_messages 
                 WHERE (sender_id = ? AND recipient_id = other_user_id)
                 OR (sender_id = other_user_id AND recipient_id = ?)
                 ORDER BY created_at DESC LIMIT 1) as last_message_time,
                (SELECT COUNT(*) FROM direct_messages
                 WHERE sender_id = other_user_id 
                 AND recipient_id = ? 
                 AND is_read = 0) as unread_count
            FROM direct_messages dm
            JOIN users u ON u.user_id = CASE 
                WHEN dm.sender_id = ? THEN dm.recipient_id
                ELSE dm.sender_id
            END
            WHERE (dm.sender_id = ? OR dm.recipient_id = ?)
            AND dm.is_deleted = 0
            ORDER BY last_message_time DESC
        ''', (self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id']))
        
        conversations = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return conversations
    
    def create_notification(self, user_id: int, notif_type: str, content: str,
                          related_user_id: int = None, related_tweet_id: int = None):
        """Create a notification"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT INTO notifications (user_id, type, content, related_user_id, related_tweet_id)
            VALUES (?, ?, ?, ?, ?)
        ''', (user_id, notif_type, content, related_user_id, related_tweet_id))
        
        conn.commit()
        conn.close()
    
    def get_notifications(self, limit: int = 50) -> List[Dict]:
        """Get user notifications"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT n.*,
                   u.username as related_username,
                   u.profile_image as related_user_image
            FROM notifications n
            LEFT JOIN users u ON n.related_user_id = u.user_id
            WHERE n.user_id = ?
            ORDER BY n.created_at DESC
            LIMIT ?
        ''', (self.current_user['user_id'], limit))
        
        notifications = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return notifications
    
    def mark_notifications_read(self) -> Tuple[bool, str]:
        """Mark all notifications as read"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            UPDATE notifications SET is_read = 1 WHERE user_id = ? AND is_read = 0
        ''', (self.current_user['user_id'],))
        
        conn.commit()
        conn.close()
        return True, "Notifications marked as read"
    
    def get_unread_notification_count(self) -> int:
        """Get count of unread notifications"""
        if not self.current_user:
            return 0
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT COUNT(*) as count FROM notifications 
            WHERE user_id = ? AND is_read = 0
        ''', (self.current_user['user_id'],))
        
        result = cursor.fetchone()
        conn.close()
        return result['count'] if result else 0
    
    def create_list(self, name: str, description: str = "", 
                   is_private: bool = False) -> Tuple[bool, str, int]:
        """Create a new list"""
        if not self.current_user:
            return False, "Please login first", 0
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT INTO lists (user_id, name, description, is_private)
            VALUES (?, ?, ?, ?)
        ''', (self.current_user['user_id'], name, description, 1 if is_private else 0))
        
        list_id = cursor.lastrowid
        conn.commit()
        conn.close()
        
        return True, "List created", list_id
    
    def add_to_list(self, list_id: int, username: str) -> Tuple[bool, str]:
        """Add a user to a list"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        # Check if list belongs to current user
        cursor.execute('SELECT user_id FROM lists WHERE list_id = ?', (list_id,))
        list_owner = cursor.fetchone()
        
        if not list_owner or list_owner['user_id'] != self.current_user['user_id']:
            conn.close()
            return False, "List not found or you don't have permission"
        
        # Get target user
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        try:
            cursor.execute('''
                INSERT INTO list_members (list_id, user_id) VALUES (?, ?)
            ''', (list_id, target_user['user_id']))
            conn.commit()
            conn.close()
            return True, f"Added {username} to list"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "User already in list"
    
    def remove_from_list(self, list_id: int, username: str) -> Tuple[bool, str]:
        """Remove a user from a list"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id FROM lists WHERE list_id = ?', (list_id,))
        list_owner = cursor.fetchone()
        
        if not list_owner or list_owner['user_id'] != self.current_user['user_id']:
            conn.close()
            return False, "List not found or you don't have permission"
        
        cursor.execute('SELECT user_id FROM users WHERE username = ?', (username,))
        target_user = cursor.fetchone()
        
        if not target_user:
            conn.close()
            return False, "User not found"
        
        cursor.execute('''
            DELETE FROM list_members WHERE list_id = ? AND user_id = ?
        ''', (list_id, target_user['user_id']))
        
        conn.commit()
        conn.close()
        return True, f"Removed {username} from list"
    
    def get_user_lists(self, username: str = None) -> List[Dict]:
        """Get lists owned by a user"""
        if username is None and self.current_user:
            username = self.current_user['username']
        
        if not username:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT l.*,
                   (SELECT COUNT(*) FROM list_members WHERE list_id = l.list_id) as member_count
            FROM lists l
            JOIN users u ON l.user_id = u.user_id
            WHERE u.username = ?
            ORDER BY l.created_at DESC
        ''', (username,))
        
        lists = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return lists
    
    def get_list_timeline(self, list_id: int, limit: int = 50) -> List[Dict]:
        """Get timeline for a specific list"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT t.*, u.username, u.full_name, u.profile_image, u.verified,
                   (SELECT COUNT(*) FROM likes WHERE tweet_id = t.tweet_id) as like_count,
                   (SELECT COUNT(*) FROM retweets WHERE tweet_id = t.tweet_id) as retweet_count,
                   (SELECT COUNT(*) FROM tweets WHERE reply_to = t.tweet_id AND is_deleted = 0) as reply_count
            FROM tweets t
            JOIN users u ON t.user_id = u.user_id
            JOIN list_members lm ON u.user_id = lm.user_id
            WHERE lm.list_id = ? AND t.is_deleted = 0
            ORDER BY t.created_at DESC
            LIMIT ?
        ''', (list_id, limit))
        
        tweets = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return tweets
    
    def create_poll(self, tweet_content: str, options: List[str], 
                   duration_hours: int = 24) -> Tuple[bool, str, int]:
        """Create a poll tweet"""
        if not self.current_user:
            return False, "Please login first", 0
        
        if len(options) < 2 or len(options) > 4:
            return False, "Poll must have 2-4 options", 0
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        try:
            # Create tweet
            cursor.execute('''
                INSERT INTO tweets (user_id, content) VALUES (?, ?)
            ''', (self.current_user['user_id'], tweet_content))
            
            tweet_id = cursor.lastrowid
            
            # Create poll
            ends_at = datetime.now() + timedelta(hours=duration_hours)
            cursor.execute('''
                INSERT INTO polls (tweet_id, duration_hours, ends_at)
                VALUES (?, ?, ?)
            ''', (tweet_id, duration_hours, ends_at))
            
            poll_id = cursor.lastrowid
            
            # Update tweet with poll_id
            cursor.execute('UPDATE tweets SET poll_id = ? WHERE tweet_id = ?', 
                         (poll_id, tweet_id))
            
            # Add poll options
            for option in options:
                cursor.execute('''
                    INSERT INTO poll_options (poll_id, option_text) VALUES (?, ?)
                ''', (poll_id, option))
            
            conn.commit()
            conn.close()
            return True, "Poll created", tweet_id
        except Exception as e:
            conn.close()
            return False, str(e), 0
    
    def vote_poll(self, poll_id: int, option_id: int) -> Tuple[bool, str]:
        """Vote in a poll"""
        if not self.current_user:
            return False, "Please login first"
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        # Check if poll is still active
        cursor.execute('SELECT ends_at FROM polls WHERE poll_id = ?', (poll_id,))
        poll = cursor.fetchone()
        
        if not poll:
            conn.close()
            return False, "Poll not found"
        
        if datetime.fromisoformat(poll['ends_at']) < datetime.now():
            conn.close()
            return False, "Poll has ended"
        
        try:
            # Record vote
            cursor.execute('''
                INSERT INTO poll_votes (poll_id, option_id, user_id)
                VALUES (?, ?, ?)
            ''', (poll_id, option_id, self.current_user['user_id']))
            
            # Increment vote count
            cursor.execute('''
                UPDATE poll_options SET votes = votes + 1 WHERE option_id = ?
            ''', (option_id,))
            
            conn.commit()
            conn.close()
            return True, "Vote recorded"
        except sqlite3.IntegrityError:
            conn.close()
            return False, "You've already voted in this poll"
    
    def get_poll_results(self, poll_id: int) -> Dict:
        """Get poll results"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT p.*,
                   (SELECT COUNT(*) FROM poll_votes WHERE poll_id = p.poll_id) as total_votes,
                   (p.ends_at < datetime('now')) as has_ended
            FROM polls p
            WHERE p.poll_id = ?
        ''', (poll_id,))
        
        poll = cursor.fetchone()
        
        if not poll:
            conn.close()
            return None
        
        poll_dict = dict(poll)
        
        cursor.execute('''
            SELECT * FROM poll_options WHERE poll_id = ?
        ''', (poll_id,))
        
        poll_dict['options'] = [dict(row) for row in cursor.fetchall()]
        
        # Check if current user voted
        if self.current_user:
            cursor.execute('''
                SELECT option_id FROM poll_votes 
                WHERE poll_id = ? AND user_id = ?
            ''', (poll_id, self.current_user['user_id']))
            
            vote = cursor.fetchone()
            poll_dict['user_vote'] = vote['option_id'] if vote else None
        
        conn.close()
        return poll_dict
    
    def get_who_to_follow(self, limit: int = 5) -> List[Dict]:
        """Get suggested users to follow"""
        if not self.current_user:
            return []
        
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT u.*,
                   (SELECT COUNT(*) FROM follows WHERE following_id = u.user_id) as followers_count,
                   (SELECT COUNT(*) FROM tweets WHERE user_id = u.user_id AND is_deleted = 0) as tweet_count
            FROM users u
            WHERE u.user_id != ?
            AND u.user_id NOT IN (
                SELECT following_id FROM follows WHERE follower_id = ?
            )
            AND u.user_id NOT IN (
                SELECT blocked_id FROM blocks WHERE blocker_id = ?
            )
            ORDER BY followers_count DESC, tweet_count DESC
            LIMIT ?
        ''', (self.current_user['user_id'], self.current_user['user_id'],
              self.current_user['user_id'], limit))
        
        suggestions = [dict(row) for row in cursor.fetchall()]
        conn.close()
        return suggestions
