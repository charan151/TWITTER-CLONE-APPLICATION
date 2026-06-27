# twitter.py - Complete Twitter Clone
# Run with: python twitter.py
# No external libraries needed!

import json
import os
import hashlib
import re
from datetime import datetime

# ============================================================
# DATA MANAGER CLASS
# ============================================================
class DataManager:
    def __init__(self):
        self.data_file = 'twitter_data.json'
        self.data = self.load_data()
    
    def load_data(self):
        if os.path.exists(self.data_file):
            try:
                with open(self.data_file, 'r') as f:
                    return json.load(f)
            except:
                return self.default_data()
        return self.default_data()
    
    def default_data(self):
        return {
            'users': {},
            'tweets': {},
            'notifications': {},
            'messages': {},
            'tweet_counter': 0
        }
    
    def save(self):
        with open(self.data_file, 'w') as f:
            json.dump(self.data, f, indent=2)

# ============================================================
# TWITTER CLONE CLASS
# ============================================================
class TwitterClone:
    def __init__(self):
        self.dm = DataManager()
        self.current_user = None
    
    # ==================== HELPER METHODS ====================
    
    @staticmethod
    def hash_password(password):
        return hashlib.sha256(password.encode()).hexdigest()
    
    @staticmethod
    def get_time():
        return datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    
    @staticmethod
    def time_ago(timestamp):
        try:
            dt = datetime.strptime(timestamp, '%Y-%m-%d %H:%M:%S')
            diff = datetime.now() - dt
            seconds = diff.total_seconds()
            
            if seconds < 60:
                return f"{int(seconds)}s ago"
            elif seconds < 3600:
                return f"{int(seconds / 60)}m ago"
            elif seconds < 86400:
                return f"{int(seconds / 3600)}h ago"
            elif seconds < 604800:
                return f"{int(seconds / 86400)}d ago"
            else:
                return dt.strftime("%b %d, %Y")
        except:
            return timestamp
    
    @staticmethod
    def extract_hashtags(content):
        return re.findall(r'#(\w+)', content)
    
    @staticmethod
    def extract_mentions(content):
        return re.findall(r'@(\w+)', content)
    
    def get_tweet_id(self):
        self.dm.data['tweet_counter'] += 1
        self.dm.save()
        return str(self.dm.data['tweet_counter'])
    
    def clear_screen(self):
        os.system('cls' if os.name == 'nt' else 'clear')
    
    def print_header(self, title="TWITTER CLONE"):
        print("\n" + "="*65)
        print(f" 🐦 {title} ".center(65, "="))
        print("="*65)
    
    def print_tweet(self, tweet, index=None):
        verified = "✓" if self.dm.data['users'].get(
            tweet['username'], {}).get('verified', False) else ""
        prefix = f"[{index}] " if index else ""
        print(f"\n{prefix}@{tweet['username']}{verified}  •  {self.time_ago(tweet['timestamp'])}")
        print(f"    {tweet['content']}")
        
        if tweet.get('image'):
            print(f"    🖼️  [Image: {tweet['image']}]")
        
        if tweet.get('hashtags'):
            print(f"    🏷️  {' '.join(['#'+h for h in tweet['hashtags']])}")
        
        print(f"    ❤️  {len(tweet.get('likes', []))}  "
              f"🔄 {len(tweet.get('retweets', []))}  "
              f"💬 {len(tweet.get('replies', []))}  "
              f"🔖 {len(tweet.get('bookmarks', []))}  "
              f"ID: #{tweet['id']}")
        print("    " + "-"*55)
    
    def print_user(self, username):
        user = self.dm.data['users'].get(username)
        if not user:
            return
        
        verified = "✓ Verified" if user.get('verified') else ""
        print(f"\n  👤 @{username} {verified}")
        print(f"     Name: {user.get('full_name') or 'Not set'}")
        print(f"     Bio: {user.get('bio') or 'No bio'}")
        print(f"     Location: {user.get('location') or 'Not set'}")
        print(f"     Website: {user.get('website') or 'Not set'}")
        print(f"     Tweets: {len([t for t in self.dm.data['tweets'].values() if t['username'] == username])}")
        print(f"     Followers: {len(user.get('followers', []))}")
        print(f"     Following: {len(user.get('following', []))}")
        print(f"     Joined: {user.get('created_at', 'Unknown')[:10]}")
        print("     " + "-"*50)
    
    def add_notification(self, username, notif_type, message, related_tweet_id=None):
        if username not in self.dm.data['notifications']:
            self.dm.data['notifications'][username] = []
        
        self.dm.data['notifications'][username].append({
            'type': notif_type,
            'message': message,
            'timestamp': self.get_time(),
            'is_read': False,
            'related_tweet_id': related_tweet_id
        })
        self.dm.save()
    
    def get_input(self, prompt, max_length=None, allow_empty=False):
        while True:
            value = input(prompt).strip()
            if not value and not allow_empty:
                print("❌ This field cannot be empty. Please try again.")
                continue
            if max_length and len(value) > max_length:
                print(f"❌ Input too long. Maximum {max_length} characters.")
                continue
            return value
    
    # ==================== AUTH METHODS ====================
    
    def register(self):
        self.print_header("REGISTER")
        print("\nCreate your Twitter Clone account\n")
        
        username = self.get_input("Username: ")
        
        if username in self.dm.data['users']:
            print("\n❌ Username already exists!")
            return False
        
        if not re.match(r'^[a-zA-Z0-9_]+$', username):
            print("\n❌ Username can only contain letters, numbers, and underscores!")
            return False
        
        email = self.get_input("Email: ")
        password = self.get_input("Password (min 6 chars): ")
        
        if len(password) < 6:
            print("\n❌ Password must be at least 6 characters!")
            return False
        
        full_name = self.get_input("Full Name: ", allow_empty=True)
        
        # Check email already used
        for user in self.dm.data['users'].values():
            if user.get('email') == email:
                print("\n❌ Email already in use!")
                return False
        
        self.dm.data['users'][username] = {
            'email': email,
            'password': self.hash_password(password),
            'full_name': full_name,
            'bio': '',
            'location': '',
            'website': '',
            'profile_image': '🐦',
            'followers': [],
            'following': [],
            'blocked': [],
            'muted': [],
            'verified': False,
            'created_at': self.get_time()
        }
        self.dm.save()
        print("\n✅ Registration successful! You can now login.")
        return True
    
    def login(self):
        self.print_header("LOGIN")
        print("\nLogin to your account\n")
        
        username = self.get_input("Username: ")
        password = self.get_input("Password: ")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return False
        
        if self.dm.data['users'][username]['password'] != self.hash_password(password):
            print("\n❌ Invalid password!")
            return False
        
        self.current_user = username
        user = self.dm.data['users'][username]
        
        # Count unread notifications
        notifs = self.dm.data['notifications'].get(username, [])
        unread = len([n for n in notifs if not n['is_read']])
        
        print(f"\n✅ Welcome back, @{username}!")
        if unread > 0:
            print(f"🔔 You have {unread} unread notifications!")
        return True
    
    def logout(self):
        self.current_user = None
        print("\n✅ Logged out successfully!")
    
    def change_password(self):
        self.print_header("CHANGE PASSWORD")
        
        old_password = self.get_input("Current Password: ")
        if self.dm.data['users'][self.current_user]['password'] != self.hash_password(old_password):
            print("\n❌ Incorrect current password!")
            return
        
        new_password = self.get_input("New Password (min 6 chars): ")
        if len(new_password) < 6:
            print("\n❌ Password too short!")
            return
        
        confirm_password = self.get_input("Confirm New Password: ")
        if new_password != confirm_password:
            print("\n❌ Passwords don't match!")
            return
        
        self.dm.data['users'][self.current_user]['password'] = self.hash_password(new_password)
        self.dm.save()
        print("\n✅ Password changed successfully!")
    
    # ==================== PROFILE METHODS ====================
    
    def view_profile(self, username=None):
        if not username:
            username = self.current_user
        
        self.print_header(f"@{username}'s PROFILE")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        self.print_user(username)
        
        # Show follow status
        if username != self.current_user:
            user = self.dm.data['users'][self.current_user]
            if username in user['following']:
                print(f"\n  ✅ You are following @{username}")
            if username in user.get('blocked', []):
                print(f"\n  🚫 You have blocked @{username}")
            if username in user.get('muted', []):
                print(f"\n  🔇 You have muted @{username}")
        
        # Show tweets
        user_tweets = [t for t in self.dm.data['tweets'].values()
                      if t['username'] == username and not t.get('is_deleted')]
        user_tweets.sort(key=lambda x: x['timestamp'], reverse=True)
        
        if user_tweets:
            print(f"\n  📊 Recent Tweets ({len(user_tweets)} total):")
            for tweet in user_tweets[:10]:
                self.print_tweet(tweet)
        else:
            print("\n  📭 No tweets yet!")
    
    def edit_profile(self):
        self.print_header("EDIT PROFILE")
        user = self.dm.data['users'][self.current_user]
        
        print(f"\n1. Full Name (current: {user.get('full_name') or 'Not set'})")
        print(f"2. Bio (current: {user.get('bio') or 'Not set'})")
        print(f"3. Location (current: {user.get('location') or 'Not set'})")
        print(f"4. Website (current: {user.get('website') or 'Not set'})")
        print(f"5. Email (current: {user.get('email')})")
        print("0. Back")
        
        choice = input("\nWhat to edit? ").strip()
        
        if choice == "1":
            value = self.get_input("New Full Name: ", allow_empty=True)
            self.dm.data['users'][self.current_user]['full_name'] = value
            print("\n✅ Full name updated!")
        elif choice == "2":
            value = self.get_input("New Bio (max 160 chars): ", max_length=160, allow_empty=True)
            self.dm.data['users'][self.current_user]['bio'] = value
            print("\n✅ Bio updated!")
        elif choice == "3":
            value = self.get_input("New Location: ", allow_empty=True)
            self.dm.data['users'][self.current_user]['location'] = value
            print("\n✅ Location updated!")
        elif choice == "4":
            value = self.get_input("New Website: ", allow_empty=True)
            self.dm.data['users'][self.current_user]['website'] = value
            print("\n✅ Website updated!")
        elif choice == "5":
            value = self.get_input("New Email: ")
            # Check if email already used
            for uname, udata in self.dm.data['users'].items():
                if uname != self.current_user and udata.get('email') == value:
                    print("\n❌ Email already in use!")
                    return
            self.dm.data['users'][self.current_user]['email'] = value
            print("\n✅ Email updated!")
        
        if choice in ['1', '2', '3', '4', '5']:
            self.dm.save()
    
    # ==================== TWEET METHODS ====================
    
    def post_tweet(self, reply_to=None):
        self.print_header("POST TWEET" if not reply_to else "POST REPLY")
        
        if reply_to:
            tweet = self.dm.data['tweets'].get(reply_to)
            if tweet:
                print(f"\nReplying to @{tweet['username']}:")
                print(f"  \"{tweet['content']}\"\n")
        
        print(f"Characters remaining will be shown (max 280)\n")
        content = self.get_input("What's happening? > ", max_length=280)
        
        print(f"\nCharacters used: {len(content)}/280")
        
        # Ask for image URL
        image = input("Image URL (optional, press Enter to skip): ").strip()
        
        # Extract hashtags and mentions
        hashtags = self.extract_hashtags(content)
        mentions = self.extract_mentions(content)
        
        # Create tweet
        tweet_id = self.get_tweet_id()
        tweet = {
            'id': tweet_id,
            'username': self.current_user,
            'content': content,
            'image': image if image else None,
            'hashtags': hashtags,
            'mentions': mentions,
            'timestamp': self.get_time(),
            'likes': [],
            'retweets': [],
            'replies': [],
            'bookmarks': [],
            'is_deleted': False,
            'reply_to': reply_to
        }
        
        self.dm.data['tweets'][tweet_id] = tweet
        
        # Handle mentions - notify mentioned users
        for mention in mentions:
            if mention in self.dm.data['users'] and mention != self.current_user:
                self.add_notification(
                    mention, 'mention',
                    f"@{self.current_user} mentioned you in a tweet",
                    tweet_id
                )
        
        # Handle reply
        if reply_to and reply_to in self.dm.data['tweets']:
            self.dm.data['tweets'][reply_to]['replies'].append(tweet_id)
            original_author = self.dm.data['tweets'][reply_to]['username']
            if original_author != self.current_user:
                self.add_notification(
                    original_author, 'reply',
                    f"@{self.current_user} replied to your tweet",
                    tweet_id
                )
        
        self.dm.save()
        print(f"\n✅ Tweet posted successfully! (ID: #{tweet_id})")
        
        if hashtags:
            print(f"🏷️  Hashtags: {', '.join(['#'+h for h in hashtags])}")
        if mentions:
            print(f"👤 Mentions: {', '.join(['@'+m for m in mentions])}")
    
    def delete_tweet(self):
        self.print_header("DELETE TWEET")
        tweet_id = self.get_input("Tweet ID to delete: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        
        if tweet['username'] != self.current_user:
            print("\n❌ You can only delete your own tweets!")
            return
        
        confirm = input(f"\nAre you sure you want to delete this tweet?\n\"{tweet['content']}\"\n\nType 'yes' to confirm: ").strip().lower()
        
        if confirm == 'yes':
            self.dm.data['tweets'][tweet_id]['is_deleted'] = True
            self.dm.save()
            print("\n✅ Tweet deleted successfully!")
        else:
            print("\n❌ Deletion cancelled.")
    
    def view_tweet(self):
        self.print_header("VIEW TWEET")
        tweet_id = self.get_input("Tweet ID to view: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        if tweet.get('is_deleted'):
            print("\n❌ This tweet has been deleted!")
            return
        
        print("\n" + "="*65)
        print(" TWEET ".center(65))
        self.print_tweet(tweet)
        
        # Show original tweet if reply
        if tweet.get('reply_to'):
            original = self.dm.data['tweets'].get(tweet['reply_to'])
            if original and not original.get('is_deleted'):
                print("\n  📩 Replying to:")
                self.print_tweet(original)
        
        # Show replies
        if tweet.get('replies'):
            replies = [self.dm.data['tweets'].get(rid) for rid in tweet['replies']]
            replies = [r for r in replies if r and not r.get('is_deleted')]
            
            if replies:
                print(f"\n  💬 Replies ({len(replies)}):")
                for reply in replies:
                    self.print_tweet(reply)
        
        # Options
        print("\n  Options:")
        print("  1. ❤️  Like/Unlike")
        print("  2. 🔄 Retweet/Unretweet")
        print("  3. 💬 Reply")
        print("  4. 🔖 Bookmark/Unbookmark")
        print("  5. 📊 View Likers")
        print("  6. 📊 View Retweeters")
        print("  0. Back")
        
        choice = input("\n  Choice: ").strip()
        
        if choice == "1":
            self.like_tweet(tweet_id)
        elif choice == "2":
            self.retweet(tweet_id)
        elif choice == "3":
            self.post_tweet(reply_to=tweet_id)
        elif choice == "4":
            self.bookmark_tweet(tweet_id)
        elif choice == "5":
            if tweet['likes']:
                print(f"\n  ❤️  Liked by: {', '.join(['@'+u for u in tweet['likes']])}")
            else:
                print("\n  No likes yet")
        elif choice == "6":
            if tweet['retweets']:
                print(f"\n  🔄 Retweeted by: {', '.join(['@'+u for u in tweet['retweets']])}")
            else:
                print("\n  No retweets yet")
    
    def like_tweet(self, tweet_id=None):
        if not tweet_id:
            self.print_header("LIKE TWEET")
            tweet_id = self.get_input("Tweet ID to like/unlike: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        
        if tweet.get('is_deleted'):
            print("\n❌ This tweet has been deleted!")
            return
        
        if self.current_user in tweet['likes']:
            tweet['likes'].remove(self.current_user)
            print(f"\n💔 Tweet unliked! ({len(tweet['likes'])} likes)")
        else:
            tweet['likes'].append(self.current_user)
            print(f"\n❤️  Tweet liked! ({len(tweet['likes'])} likes)")
            
            if tweet['username'] != self.current_user:
                self.add_notification(
                    tweet['username'], 'like',
                    f"@{self.current_user} liked your tweet",
                    tweet_id
                )
        
        self.dm.save()
    
    def retweet(self, tweet_id=None):
        if not tweet_id:
            self.print_header("RETWEET")
            tweet_id = self.get_input("Tweet ID to retweet/unretweet: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        
        if tweet.get('is_deleted'):
            print("\n❌ This tweet has been deleted!")
            return
        
        if self.current_user in tweet['retweets']:
            tweet['retweets'].remove(self.current_user)
            print(f"\n↩️  Retweet removed! ({len(tweet['retweets'])} retweets)")
        else:
            tweet['retweets'].append(self.current_user)
            print(f"\n🔄 Retweeted! ({len(tweet['retweets'])} retweets)")
            
            if tweet['username'] != self.current_user:
                self.add_notification(
                    tweet['username'], 'retweet',
                    f"@{self.current_user} retweeted your tweet",
                    tweet_id
                )
        
        self.dm.save()
    
    def bookmark_tweet(self, tweet_id=None):
        if not tweet_id:
            self.print_header("BOOKMARK TWEET")
            tweet_id = self.get_input("Tweet ID to bookmark: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        
        if self.current_user in tweet.get('bookmarks', []):
            tweet['bookmarks'].remove(self.current_user)
            print("\n🗑️  Bookmark removed!")
        else:
            if 'bookmarks' not in tweet:
                tweet['bookmarks'] = []
            tweet['bookmarks'].append(self.current_user)
            print("\n🔖 Tweet bookmarked!")
        
        self.dm.save()
    
    def quote_tweet(self):
        self.print_header("QUOTE TWEET")
        tweet_id = self.get_input("Tweet ID to quote: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        original = self.dm.data['tweets'][tweet_id]
        print(f"\nQuoting @{original['username']}: \"{original['content']}\"\n")
        
        content = self.get_input("Your comment (max 280 chars): ", max_length=280)
        full_content = f"{content}\n\n💬 Quoting @{original['username']}: \"{original['content']}\""
        
        tweet_id_new = self.get_tweet_id()
        tweet = {
            'id': tweet_id_new,
            'username': self.current_user,
            'content': full_content,
            'image': None,
            'hashtags': self.extract_hashtags(content),
            'mentions': self.extract_mentions(content),
            'timestamp': self.get_time(),
            'likes': [],
            'retweets': [],
            'replies': [],
            'bookmarks': [],
            'is_deleted': False,
            'reply_to': None,
            'quote_of': tweet_id
        }
        
        self.dm.data['tweets'][tweet_id_new] = tweet
        
        if original['username'] != self.current_user:
            self.add_notification(
                original['username'], 'quote',
                f"@{self.current_user} quoted your tweet",
                tweet_id_new
            )
        
        self.dm.save()
        print(f"\n✅ Quote tweet posted! (ID: #{tweet_id_new})")
    
    # ==================== TIMELINE METHODS ====================
    
    def view_timeline(self):
        self.print_header("YOUR TIMELINE")
        
        user = self.dm.data['users'][self.current_user]
        following = user['following']
        blocked = user.get('blocked', [])
        muted = user.get('muted', [])
        
        # Get tweets from followed users + own tweets
        timeline = []
        for tweet in self.dm.data['tweets'].values():
            if (tweet['username'] in following or tweet['username'] == self.current_user):
                if not tweet.get('is_deleted'):
                    if tweet['username'] not in blocked:
                        if tweet['username'] not in muted:
                            timeline.append(tweet)
        
        if not timeline:
            print("\n📭 Your timeline is empty!")
            print("Follow some users to see their tweets here.")
            return
        
        timeline.sort(key=lambda x: x['timestamp'], reverse=True)
        
        print(f"\n📊 Showing {min(20, len(timeline))} of {len(timeline)} tweets\n")
        
        for tweet in timeline[:20]:
            self.print_tweet(tweet)
    
    def view_explore(self):
        self.print_header("EXPLORE / ALL TWEETS")
        
        user = self.dm.data['users'][self.current_user]
        blocked = user.get('blocked', [])
        muted = user.get('muted', [])
        
        # Get all public tweets
        all_tweets = [t for t in self.dm.data['tweets'].values()
                     if not t.get('is_deleted')
                     and t['username'] not in blocked
                     and t['username'] not in muted]
        
        if not all_tweets:
            print("\n📭 No tweets available yet!")
            return
        
        all_tweets.sort(key=lambda x: x['timestamp'], reverse=True)
        
        print(f"\n📊 Showing {min(20, len(all_tweets))} of {len(all_tweets)} tweets\n")
        
        for tweet in all_tweets[:20]:
            self.print_tweet(tweet)
    
    def view_trending(self):
        self.print_header("TRENDING HASHTAGS")
        
        # Count hashtags
        hashtag_count = {}
        for tweet in self.dm.data['tweets'].values():
            if not tweet.get('is_deleted'):
                for tag in tweet.get('hashtags', []):
                    tag = tag.lower()
                    hashtag_count[tag] = hashtag_count.get(tag, 0) + 1
        
        if not hashtag_count:
            print("\n📭 No trending hashtags yet!")
            return
        
        sorted_hashtags = sorted(hashtag_count.items(), key=lambda x: x[1], reverse=True)
        
        print("\n🔥 TOP TRENDING HASHTAGS:\n")
        for i, (tag, count) in enumerate(sorted_hashtags[:10], 1):
            bar = "█" * min(count * 2, 20)
            print(f"  {i:2}. #{tag:<20} {bar} ({count} tweets)")
        
        print("\n📌 View tweets by hashtag? (e.g., #python)")
        search = input("Enter hashtag (or press Enter to skip): ").strip()
        
        if search:
            tag = search.lstrip('#').lower()
            self.search_hashtag(tag)
    
    # ==================== SEARCH METHODS ====================
    
    def search_tweets(self):
        self.print_header("SEARCH TWEETS")
        query = self.get_input("Search query: ")
        
        user = self.dm.data['users'][self.current_user]
        blocked = user.get('blocked', [])
        muted = user.get('muted', [])
        
        results = []
        for tweet in self.dm.data['tweets'].values():
            if (query.lower() in tweet['content'].lower()
                    and not tweet.get('is_deleted')
                    and tweet['username'] not in blocked
                    and tweet['username'] not in muted):
                results.append(tweet)
        
        if not results:
            print(f"\n🔍 No tweets found for '{query}'")
            return
        
        results.sort(key=lambda x: x['timestamp'], reverse=True)
        print(f"\n🔍 Found {len(results)} tweets for '{query}':\n")
        
        for tweet in results[:20]:
            self.print_tweet(tweet)
    
    def search_users(self):
        self.print_header("SEARCH USERS")
        query = self.get_input("Search username or name: ")
        
        results = []
        for username, user in self.dm.data['users'].items():
            if (query.lower() in username.lower() or
                    query.lower() in user.get('full_name', '').lower()):
                results.append(username)
        
        if not results:
            print(f"\n🔍 No users found for '{query}'")
            return
        
        print(f"\n🔍 Found {len(results)} users:\n")
        for username in results:
            self.print_user(username)
    
    def search_hashtag(self, tag=None):
        self.print_header("SEARCH HASHTAG")
        
        if not tag:
            tag = self.get_input("Hashtag (without #): ").lower()
        
        results = []
        for tweet in self.dm.data['tweets'].values():
            if (tag in [h.lower() for h in tweet.get('hashtags', [])]
                    and not tweet.get('is_deleted')):
                results.append(tweet)
        
        if not results:
            print(f"\n🔍 No tweets found for #{tag}")
            return
        
        results.sort(key=lambda x: x['timestamp'], reverse=True)
        print(f"\n🔍 Found {len(results)} tweets for #{tag}:\n")
        
        for tweet in results[:20]:
            self.print_tweet(tweet)
    
    # ==================== SOCIAL METHODS ====================
    
    def follow_user(self):
        self.print_header("FOLLOW USER")
        username = self.get_input("Username to follow: ")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        if username == self.current_user:
            print("\n❌ You cannot follow yourself!")
            return
        
        user = self.dm.data['users'][self.current_user]
        
        if username in user.get('blocked', []):
            print("\n❌ You have blocked this user. Unblock them first.")
            return
        
        if username in user['following']:
            print(f"\n❌ You are already following @{username}!")
            return
        
        self.dm.data['users'][self.current_user]['following'].append(username)
        self.dm.data['users'][username]['followers'].append(self.current_user)
        
        self.add_notification(
            username, 'follow',
            f"@{self.current_user} started following you"
        )
        
        self.dm.save()
        print(f"\n✅ You are now following @{username}!")
    
    def unfollow_user(self):
        self.print_header("UNFOLLOW USER")
        
        following = self.dm.data['users'][self.current_user]['following']
        
        if not following:
            print("\n📭 You are not following anyone!")
            return
        
        print("\nUsers you follow:")
        for i, username in enumerate(following, 1):
            print(f"  {i}. @{username}")
        
        username = self.get_input("\nUsername to unfollow: ")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        if username not in following:
            print(f"\n❌ You are not following @{username}!")
            return
        
        self.dm.data['users'][self.current_user]['following'].remove(username)
        if self.current_user in self.dm.data['users'][username]['followers']:
            self.dm.data['users'][username]['followers'].remove(self.current_user)
        
        self.dm.save()
        print(f"\n✅ Unfollowed @{username} successfully!")
    
    def view_followers(self):
        self.print_header("MY FOLLOWERS")
        
        followers = self.dm.data['users'][self.current_user]['followers']
        
        if not followers:
            print("\n📭 You have no followers yet!")
            return
        
        print(f"\n👥 Your Followers ({len(followers)}):\n")
        for username in followers:
            self.print_user(username)
    
    def view_following(self):
        self.print_header("USERS I FOLLOW")
        
        following = self.dm.data['users'][self.current_user]['following']
        
        if not following:
            print("\n📭 You are not following anyone yet!")
            return
        
        print(f"\n👥 Following ({len(following)}):\n")
        for username in following:
            self.print_user(username)
    
    def block_user(self):
        self.print_header("BLOCK USER")
        username = self.get_input("Username to block: ")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        if username == self.current_user:
            print("\n❌ You cannot block yourself!")
            return
        
        user = self.dm.data['users'][self.current_user]
        
        if username in user.get('blocked', []):
            print(f"\n❌ @{username} is already blocked!")
            return
        
        # Remove follow relationships
        if username in user['following']:
            user['following'].remove(username)
            if self.current_user in self.dm.data['users'][username]['followers']:
                self.dm.data['users'][username]['followers'].remove(self.current_user)
        
        if self.current_user in self.dm.data['users'][username]['following']:
            self.dm.data['users'][username]['following'].remove(self.current_user)
            if username in user['followers']:
                user['followers'].remove(username)
        
        if 'blocked' not in user:
            user['blocked'] = []
        user['blocked'].append(username)
        
        self.dm.save()
        print(f"\n✅ @{username} has been blocked!")
    
    def unblock_user(self):
        self.print_header("UNBLOCK USER")
        
        blocked = self.dm.data['users'][self.current_user].get('blocked', [])
        
        if not blocked:
            print("\n📭 You haven't blocked anyone!")
            return
        
        print("\nBlocked users:")
        for i, username in enumerate(blocked, 1):
            print(f"  {i}. @{username}")
        
        username = self.get_input("\nUsername to unblock: ")
        
        if username not in blocked:
            print(f"\n❌ @{username} is not blocked!")
            return
        
        self.dm.data['users'][self.current_user]['blocked'].remove(username)
        self.dm.save()
        print(f"\n✅ @{username} has been unblocked!")
    
    def mute_user(self):
        self.print_header("MUTE USER")
        username = self.get_input("Username to mute: ")
        
        if username not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        if username == self.current_user:
            print("\n❌ You cannot mute yourself!")
            return
        
        user = self.dm.data['users'][self.current_user]
        
        if username in user.get('muted', []):
            print(f"\n❌ @{username} is already muted!")
            return
        
        if 'muted' not in user:
            user['muted'] = []
        user['muted'].append(username)
        
        self.dm.save()
        print(f"\n✅ @{username} has been muted! (You won't see their tweets)")
    
    def unmute_user(self):
        self.print_header("UNMUTE USER")
        
        muted = self.dm.data['users'][self.current_user].get('muted', [])
        
        if not muted:
            print("\n📭 You haven't muted anyone!")
            return
        
        print("\nMuted users:")
        for i, username in enumerate(muted, 1):
            print(f"  {i}. @{username}")
        
        username = self.get_input("\nUsername to unmute: ")
        
        if username not in muted:
            print(f"\n❌ @{username} is not muted!")
            return
        
        self.dm.data['users'][self.current_user]['muted'].remove(username)
        self.dm.save()
        print(f"\n✅ @{username} has been unmuted!")
    
    def suggested_users(self):
        self.print_header("WHO TO FOLLOW")
        
        user = self.dm.data['users'][self.current_user]
        following = user['following']
        blocked = user.get('blocked', [])
        
        # Get users not followed
        suggestions = []
        for username, udata in self.dm.data['users'].items():
            if (username != self.current_user
                    and username not in following
                    and username not in blocked):
                tweet_count = len([t for t in self.dm.data['tweets'].values()
                                  if t['username'] == username and not t.get('is_deleted')])
                suggestions.append((username, len(udata['followers']), tweet_count))
        
        if not suggestions:
            print("\n📭 No suggestions available!")
            return
        
        # Sort by followers
        suggestions.sort(key=lambda x: x[1], reverse=True)
        
        print("\n👥 SUGGESTED USERS TO FOLLOW:\n")
        for username, followers, tweets in suggestions[:10]:
            print(f"  👤 @{username}")
            print(f"     Followers: {followers} | Tweets: {tweets}")
            name = self.dm.data['users'][username].get('full_name')
            bio = self.dm.data['users'][username].get('bio')
            if name:
                print(f"     Name: {name}")
            if bio:
                print(f"     Bio: {bio}")
            print()
    
    # ==================== MESSAGES METHODS ====================
    
    def send_message(self):
        self.print_header("SEND MESSAGE")
        recipient = self.get_input("Recipient username: ")
        
        if recipient not in self.dm.data['users']:
            print("\n❌ User not found!")
            return
        
        if recipient == self.current_user:
            print("\n❌ You cannot message yourself!")
            return
        
        # Check if blocked
        target_user = self.dm.data['users'][recipient]
        current_user = self.dm.data['users'][self.current_user]
        
        if self.current_user in target_user.get('blocked', []):
            print("\n❌ You cannot message this user!")
            return
        
        if recipient in current_user.get('blocked', []):
            print("\n❌ You have blocked this user!")
            return
        
        content = self.get_input("Message: ", max_length=1000)
        
        # Create conversation key
        conv_key = '_'.join(sorted([self.current_user, recipient]))
        
        if conv_key not in self.dm.data['messages']:
            self.dm.data['messages'][conv_key] = []
        
        self.dm.data['messages'][conv_key].append({
            'sender': self.current_user,
            'recipient': recipient,
            'content': content,
            'timestamp': self.get_time(),
            'is_read': False
        })
        
        self.add_notification(
            recipient, 'message',
            f"@{self.current_user} sent you a message"
        )
        
        self.dm.save()
        print(f"\n✅ Message sent to @{recipient}!")
    
    def view_messages(self):
        self.print_header("MESSAGES")
        
        # Find all conversations
        conversations = []
        for conv_key, messages in self.dm.data['messages'].items():
            participants = conv_key.split('_')
            if self.current_user in participants:
                other = [p for p in participants if p != self.current_user][0]
                unread = len([m for m in messages
                             if m['recipient'] == self.current_user and not m['is_read']])
                last_msg = messages[-1] if messages else None
                conversations.append((other, unread, last_msg, conv_key))
        
        if not conversations:
            print("\n📭 No messages yet!")
            return
        
        # Sort by last message time
        conversations.sort(key=lambda x: x[2]['timestamp'] if x[2] else '', reverse=True)
        
        print("\n💬 YOUR CONVERSATIONS:\n")
        for i, (other, unread, last_msg, _) in enumerate(conversations, 1):
            unread_badge = f" 🔴 {unread} unread" if unread > 0 else ""
            print(f"  {i}. @{other}{unread_badge}")
            if last_msg:
                sender = "You" if last_msg['sender'] == self.current_user else f"@{last_msg['sender']}"
                preview = last_msg['content'][:40] + "..." if len(last_msg['content']) > 40 else last_msg['content']
                print(f"     {sender}: {preview}")
                print(f"     {self.time_ago(last_msg['timestamp'])}")
            print()
        
        choice = input("Enter number to open conversation (or 0 to go back): ").strip()
        
        if choice.isdigit() and 1 <= int(choice) <= len(conversations):
            other = conversations[int(choice)-1][0]
            self.view_conversation(other)
    
    def view_conversation(self, other_username):
        self.print_header(f"CONVERSATION WITH @{other_username}")
        
        conv_key = '_'.join(sorted([self.current_user, other_username]))
        
        if conv_key not in self.dm.data['messages']:
            print("\n📭 No messages yet in this conversation!")
        else:
            messages = self.dm.data['messages'][conv_key]
            
            print(f"\n{'='*65}\n")
            for msg in messages[-20:]:  # Show last 20 messages
                if msg['sender'] == self.current_user:
                    # Right align own messages
                    print(f"  {'You':>20}: {msg['content']}")
                    print(f"  {self.time_ago(msg['timestamp']):>20}  ✓{'✓' if msg['is_read'] else ''}\n")
                else:
                    # Left align other's messages
                    print(f"  @{other_username}: {msg['content']}")
                    print(f"  {self.time_ago(msg['timestamp'])}\n")
                
                # Mark as read
                if msg['recipient'] == self.current_user:
                    msg['is_read'] = True
            
            self.dm.save()
        
        # Options
        print("\n" + "="*65)
        print("1. 💬 Send Message")
        print("0. 🔙 Back")
        
        choice = input("\nChoice: ").strip()
        
        if choice == "1":
            content = self.get_input("Message: ", max_length=1000)
            
            conv_key = '_'.join(sorted([self.current_user, other_username]))
            
            if conv_key not in self.dm.data['messages']:
                self.dm.data['messages'][conv_key] = []
            
            self.dm.data['messages'][conv_key].append({
                'sender': self.current_user,
                'recipient': other_username,
                'content': content,
                'timestamp': self.get_time(),
                'is_read': False
            })
            
            self.add_notification(
                other_username, 'message',
                f"@{self.current_user} sent you a message"
            )
            
            self.dm.save()
            print(f"\n✅ Message sent!")
    
    # ==================== NOTIFICATION METHODS ====================
    
    def view_notifications(self):
        self.print_header("NOTIFICATIONS")
        
        notifications = self.dm.data['notifications'].get(self.current_user, [])
        
        if not notifications:
            print("\n📭 No notifications yet!")
            return
        
        # Sort by newest first
        notifications_sorted = sorted(notifications, key=lambda x: x['timestamp'], reverse=True)
        
        unread_count = len([n for n in notifications if not n['is_read']])
        print(f"\n🔔 {unread_count} unread notifications\n")
        
        icons = {
            'like': '❤️ ',
            'retweet': '🔄',
            'reply': '💬',
            'follow': '👤',
            'mention': '📢',
            'quote': '💬',
            'message': '✉️ '
        }
        
        for notif in notifications_sorted[:20]:
            icon = icons.get(notif['type'], '🔔')
            read_status = "" if notif['is_read'] else " 🔴"
            print(f"  {icon} {notif['message']}{read_status}")
            print(f"     {self.time_ago(notif['timestamp'])}")
            if notif.get('related_tweet_id'):
                print(f"     Tweet ID: #{notif['related_tweet_id']}")
            print()
        
        # Mark all as read
        for notif in self.dm.data['notifications'].get(self.current_user, []):
            notif['is_read'] = True
        self.dm.save()
    
    # ==================== BOOKMARKS METHODS ====================
    
    def view_bookmarks(self):
        self.print_header("BOOKMARKS")
        
        bookmarked = []
        for tweet in self.dm.data['tweets'].values():
            if (self.current_user in tweet.get('bookmarks', [])
                    and not tweet.get('is_deleted')):
                bookmarked.append(tweet)
        
        if not bookmarked:
            print("\n📭 No bookmarked tweets yet!")
            return
        
        bookmarked.sort(key=lambda x: x['timestamp'], reverse=True)
        
        print(f"\n🔖 Your Bookmarks ({len(bookmarked)} tweets):\n")
        for tweet in bookmarked:
            self.print_tweet(tweet)
    
    # ==================== LISTS METHODS ====================
    
    def manage_lists(self):
        self.print_header("LISTS")
        
        user = self.dm.data['users'][self.current_user]
        if 'lists' not in user:
            user['lists'] = {}
        
        lists = user['lists']
        
        print("\n1. ➕ Create New List")
        print("2. 📋 View My Lists")
        print("3. 👥 Add User to List")
        print("4. 🗑️  Delete List")
        print("0. 🔙 Back")
        
        choice = input("\nChoice: ").strip()
        
        if choice == "1":
            list_name = self.get_input("List name: ")
            description = self.get_input("Description (optional): ", allow_empty=True)
            
            lists[list_name] = {
                'description': description,
                'members': [],
                'created_at': self.get_time()
            }
            self.dm.save()
            print(f"\n✅ List '{list_name}' created!")
        
        elif choice == "2":
            if not lists:
                print("\n📭 No lists created yet!")
                return
            
            for list_name, list_data in lists.items():
                print(f"\n  📋 {list_name}")
                print(f"     {list_data.get('description', '')}")
                print(f"     Members: {len(list_data['members'])}")
                if list_data['members']:
                    print(f"     {', '.join(['@'+m for m in list_data['members']])}")
        
        elif choice == "3":
            if not lists:
                print("\n📭 No lists yet! Create one first.")
                return
            
            print("\nYour lists:")
            for i, list_name in enumerate(lists.keys(), 1):
                print(f"  {i}. {list_name}")
            
            list_name = self.get_input("List name: ")
            
            if list_name not in lists:
                print("\n❌ List not found!")
                return
            
            username = self.get_input("Username to add: ")
            
            if username not in self.dm.data['users']:
                print("\n❌ User not found!")
                return
            
            if username in lists[list_name]['members']:
                print(f"\n❌ @{username} is already in this list!")
                return
            
            lists[list_name]['members'].append(username)
            self.dm.save()
            print(f"\n✅ @{username} added to list '{list_name}'!")
        
        elif choice == "4":
            if not lists:
                print("\n📭 No lists to delete!")
                return
            
            print("\nYour lists:")
            for i, list_name in enumerate(lists.keys(), 1):
                print(f"  {i}. {list_name}")
            
            list_name = self.get_input("List name to delete: ")
            
            if list_name not in lists:
                print("\n❌ List not found!")
                return
            
            confirm = input(f"Delete list '{list_name}'? (yes/no): ").strip().lower()
            if confirm == 'yes':
                del lists[list_name]
                self.dm.save()
                print(f"\n✅ List '{list_name}' deleted!")
    
    # ==================== POLLS METHODS ====================
    
    def create_poll(self):
        self.print_header("CREATE POLL")
        
        content = self.get_input("Poll question (max 280 chars): ", max_length=280)
        
        print("\nEnter poll options (minimum 2, maximum 4):")
        options = []
        for i in range(1, 5):
            option = input(f"Option {i} (press Enter to finish): ").strip()
            if not option:
                if len(options) < 2:
                    print("❌ Minimum 2 options required!")
                    continue
                break
            options.append(option)
        
        if len(options) < 2:
            print("\n❌ Poll needs at least 2 options!")
            return
        
        duration = input("\nPoll duration in hours (default: 24): ").strip()
        duration = int(duration) if duration.isdigit() else 24
        
        tweet_id = self.get_tweet_id()
        
        poll = {
            'id': tweet_id,
            'username': self.current_user,
            'content': f"📊 POLL: {content}",
            'hashtags': self.extract_hashtags(content),
            'mentions': [],
            'image': None,
            'timestamp': self.get_time(),
            'likes': [],
            'retweets': [],
            'replies': [],
            'bookmarks': [],
            'is_deleted': False,
            'reply_to': None,
            'is_poll': True,
            'poll': {
                'question': content,
                'options': {opt: [] for opt in options},
                'duration_hours': duration,
                'created_at': self.get_time(),
                'votes': {}
            }
        }
        
        self.dm.data['tweets'][tweet_id] = poll
        self.dm.save()
        
        print(f"\n✅ Poll created! (ID: #{tweet_id})")
    
    def vote_poll(self):
        self.print_header("VOTE IN POLL")
        tweet_id = self.get_input("Poll Tweet ID: ")
        
        if tweet_id not in self.dm.data['tweets']:
            print("\n❌ Tweet not found!")
            return
        
        tweet = self.dm.data['tweets'][tweet_id]
        
        if not tweet.get('is_poll'):
            print("\n❌ This tweet is not a poll!")
            return
        
        if tweet.get('is_deleted'):
            print("\n❌ This poll has been deleted!")
            return
        
        poll = tweet['poll']
        
        # Check if already voted
        if self.current_user in poll['votes']:
            print(f"\n❌ You have already voted in this poll!")
            voted_option = poll['votes'][self.current_user]
            print(f"Your vote: {voted_option}")
            print("\nCurrent results:")
            total_votes = sum(len(v) for v in poll['options'].values())
            for option, voters in poll['options'].items():
                percentage = (len(voters) / total_votes * 100) if total_votes > 0 else 0
                bar = "█" * int(percentage / 5)
                print(f"  {option}: {bar} {percentage:.1f}% ({len(voters)} votes)")
            return
        
        print(f"\n📊 {poll['question']}\n")
        
        options = list(poll['options'].keys())
        for i, option in enumerate(options, 1):
            print(f"  {i}. {option}")
        
        choice = input("\nYour vote (enter number): ").strip()
        
        if not choice.isdigit() or not (1 <= int(choice) <= len(options)):
            print("\n❌ Invalid choice!")
            return
        
        selected_option = options[int(choice)-1]
        
        poll['options'][selected_option].append(self.current_user)
        poll['votes'][self.current_user] = selected_option
        
        self.dm.save()
        
        print(f"\n✅ You voted for: {selected_option}")
        
        # Show results
        total_votes = sum(len(v) for v in poll['options'].values())
        print("\nCurrent results:")
        for option, voters in poll['options'].items():
            percentage = (len(voters) / total_votes * 100) if total_votes > 0 else 0
            bar = "█" * int(percentage / 5)
            print(f"  {option}: {bar} {percentage:.1f}% ({len(voters)} votes)")
    
    # ==================== STATS METHODS ====================
    
    def view_stats(self):
        self.print_header("MY STATS")
        
        user = self.dm.data['users'][self.current_user]
        
        # Get my tweets
        my_tweets = [t for t in self.dm.data['tweets'].values()
                    if t['username'] == self.current_user and not t.get('is_deleted')]
        
        # Calculate stats
        total_likes_received = sum(len(t.get('likes', [])) for t in my_tweets)
        total_retweets_received = sum(len(t.get('retweets', [])) for t in my_tweets)
        total_replies_received = sum(len(t.get('replies', [])) for t in my_tweets)
        
        most_liked = max(my_tweets, key=lambda t: len(t.get('likes', []))) if my_tweets else None
        most_retweeted = max(my_tweets, key=lambda t: len(t.get('retweets', []))) if my_tweets else None
        
        print(f"\n📊 ACCOUNT STATISTICS FOR @{self.current_user}\n")
        print(f"  👤 Followers:          {len(user['followers'])}")
        print(f"  👥 Following:          {len(user['following'])}")
        print(f"  📝 Total Tweets:       {len(my_tweets)}")
        print(f"  ❤️  Likes Received:    {total_likes_received}")
        print(f"  🔄 Retweets Received:  {total_retweets_received}")
        print(f"  💬 Replies Received:   {total_replies_received}")
        print(f"  🔖 Bookmarks:          {len([t for t in self.dm.data['tweets'].values() if self.current_user in t.get('bookmarks', [])])}")
        
        unread_notifs = len([n for n in self.dm.data['notifications'].get(self.current_user, []) if not n['is_read']])
        print(f"  🔔 Unread Notifications: {unread_notifs}")
        
        if most_liked:
            print(f"\n  🏆 Most Liked Tweet ({len(most_liked.get('likes', []))} likes):")
            print(f"     \"{most_liked['content'][:60]}...\"" if len(most_liked['content']) > 60 else f"     \"{most_liked['content']}\"")
        
        if most_retweeted:
            print(f"\n  🏆 Most Retweeted ({len(most_retweeted.get('retweets', []))} retweets):")
            print(f"     \"{most_retweeted['content'][:60]}...\"" if len(most_retweeted['content']) > 60 else f"     \"{most_retweeted['content']}\"")
    
    # ==================== MENUS ====================
    
    def main_menu_logged_out(self):
        """Menu when user is not logged in"""
        while True:
            self.clear_screen()
            self.print_header()
            print("\n  Welcome to Twitter Clone!")
            print("  Share your thoughts with the world.\n")
            print("  1. 📝 Register")
            print("  2. 🔑 Login")
            print("  0. ❌ Exit")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                print("\n  👋 Goodbye! Thanks for using Twitter Clone!\n")
                return False
            
            elif choice == "1":
                self.register()
                input("\n  Press Enter to continue...")
            
            elif choice == "2":
                success = self.login()
                if success:
                    input("\n  Press Enter to continue...")
                    return True
                input("\n  Press Enter to continue...")
    
    def tweets_menu(self):
        """Tweets sub-menu"""
        while True:
            self.clear_screen()
            self.print_header("TWEETS MENU")
            
            # Unread notification count
            notifs = self.dm.data['notifications'].get(self.current_user, [])
            unread = len([n for n in notifs if not n['is_read']])
            
            print(f"\n  Logged in as @{self.current_user}")
            if unread > 0:
                print(f"  🔔 {unread} unread notifications")
            print()
            print("  1. 📝 Post Tweet")
            print("  2. 🏠 View Timeline")
            print("  3. 🌍 Explore All Tweets")
            print("  4. 👁️  View Tweet Details")
            print("  5. 🗑️  Delete My Tweet")
            print("  6. ❤️  Like/Unlike Tweet")
            print("  7. 🔄 Retweet/Unretweet")
            print("  8. 💬 Reply to Tweet")
            print("  9. 🔖 Bookmark Tweet")
            print(" 10. 💭 Quote Tweet")
            print(" 11. 📊 Create Poll")
            print(" 12. 🗳️  Vote in Poll")
            print("  0. 🔙 Back to Main Menu")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                break
            elif choice == "1":
                self.post_tweet()
            elif choice == "2":
                self.view_timeline()
            elif choice == "3":
                self.view_explore()
            elif choice == "4":
                self.view_tweet()
            elif choice == "5":
                self.delete_tweet()
            elif choice == "6":
                self.like_tweet()
            elif choice == "7":
                self.retweet()
            elif choice == "8":
                tweet_id = self.get_input("Tweet ID to reply to: ")
                self.post_tweet(reply_to=tweet_id)
            elif choice == "9":
                self.bookmark_tweet()
            elif choice == "10":
                self.quote_tweet()
            elif choice == "11":
                self.create_poll()
            elif choice == "12":
                self.vote_poll()
            
            input("\n  Press Enter to continue...")
    
    def social_menu(self):
        """Social sub-menu"""
        while True:
            self.clear_screen()
            self.print_header("SOCIAL MENU")
            print(f"\n  Logged in as @{self.current_user}\n")
            print("  1. 👥 Follow User")
            print("  2. ❌ Unfollow User")
            print("  3. 👁️  View Followers")
            print("  4. 👁️  View Following")
            print("  5. 💡 Who to Follow (Suggestions)")
            print("  6. 🚫 Block User")
            print("  7. ✅ Unblock User")
            print("  8. 🔇 Mute User")
            print("  9. 🔊 Unmute User")
            print("  0. 🔙 Back to Main Menu")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                break
            elif choice == "1":
                self.follow_user()
            elif choice == "2":
                self.unfollow_user()
            elif choice == "3":
                self.view_followers()
            elif choice == "4":
                self.view_following()
            elif choice == "5":
                self.suggested_users()
            elif choice == "6":
                self.block_user()
            elif choice == "7":
                self.unblock_user()
            elif choice == "8":
                self.mute_user()
            elif choice == "9":
                self.unmute_user()
            
            input("\n  Press Enter to continue...")
    
    def search_menu(self):
        """Search sub-menu"""
        while True:
            self.clear_screen()
            self.print_header("SEARCH MENU")
            print(f"\n  Logged in as @{self.current_user}\n")
            print("  1. 🔍 Search Tweets")
            print("  2. 👤 Search Users")
            print("  3. #  Search by Hashtag")
            print("  4. 🔥 Trending Hashtags")
            print("  0. 🔙 Back to Main Menu")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                break
            elif choice == "1":
                self.search_tweets()
            elif choice == "2":
                self.search_users()
            elif choice == "3":
                self.search_hashtag()
            elif choice == "4":
                self.view_trending()
            
            input("\n  Press Enter to continue...")
    
    def messages_menu(self):
        """Messages sub-menu"""
        while True:
            self.clear_screen()
            self.print_header("MESSAGES MENU")
            print(f"\n  Logged in as @{self.current_user}\n")
            print("  1. 💬 View Conversations")
            print("  2. ✉️  Send New Message")
            print("  3. 📩 Message a User")
            print("  0. 🔙 Back to Main Menu")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                break
            elif choice == "1":
                self.view_messages()
            elif choice == "2":
                self.send_message()
            elif choice == "3":
                username = self.get_input("Username: ")
                self.view_conversation(username)
            
            input("\n  Press Enter to continue...")
    
    def profile_menu(self):
        """Profile sub-menu"""
        while True:
            self.clear_screen()
            self.print_header("PROFILE MENU")
            print(f"\n  Logged in as @{self.current_user}\n")
            print("  1. 👤 View My Profile")
            print("  2. ✏️  Edit Profile")
            print("  3. 🔒 Change Password")
            print("  4. 🔖 View Bookmarks")
            print("  5. 📊 View Stats")
            print("  6. 📋 Manage Lists")
            print("  7. 🔔 View Notifications")
            print("  8. 👁️  View Other Profile")
            print("  0. 🔙 Back to Main Menu")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                break
            elif choice == "1":
                self.view_profile()
            elif choice == "2":
                self.edit_profile()
            elif choice == "3":
                self.change_password()
            elif choice == "4":
                self.view_bookmarks()
            elif choice == "5":
                self.view_stats()
            elif choice == "6":
                self.manage_lists()
            elif choice == "7":
                self.view_notifications()
            elif choice == "8":
                username = self.get_input("Username to view: ")
                self.view_profile(username)
            
            input("\n  Press Enter to continue...")
    
    def main_menu_logged_in(self):
        """Main menu when user is logged in"""
        while True:
            self.clear_screen()
            self.print_header("MAIN MENU")
            
            user = self.dm.data['users'][self.current_user]
            notifs = self.dm.data['notifications'].get(self.current_user, [])
            unread_notifs = len([n for n in notifs if not n['is_read']])
            
            # Count unread messages
            unread_msgs = 0
            for conv_key, messages in self.dm.data['messages'].items():
                if self.current_user in conv_key.split('_'):
                    unread_msgs += len([m for m in messages
                                       if m['recipient'] == self.current_user and not m['is_read']])
            
            print(f"\n  👤 @{self.current_user}")
            print(f"  👥 {len(user['followers'])} Followers | {len(user['following'])} Following")
            
            if unread_notifs > 0 or unread_msgs > 0:
                print(f"\n  {'🔔 '+str(unread_notifs)+' notifications  ' if unread_notifs > 0 else ''}"
                      f"{'✉️  '+str(unread_msgs)+' messages' if unread_msgs > 0 else ''}")
            
            print("\n  1. 🐦 Tweets")
            print("  2. 👥 Social")
            print("  3. 🔍 Search")
            print(f"  4. ✉️  Messages {'🔴' if unread_msgs > 0 else ''}")
            print(f"  5. 👤 Profile & Settings")
            print(f"  6. 🔔 Notifications {'🔴'+str(unread_notifs) if unread_notifs > 0 else ''}")
            print("  7. 🔥 Trending")
            print("  8. 🌍 Explore")
            print("  9. 🚪 Logout")
            print("  0. ❌ Exit")
            print("\n" + "="*65)
            
            choice = input("\n  👉 Enter choice: ").strip()
            
            if choice == "0":
                print("\n  👋 Goodbye! Thanks for using Twitter Clone!\n")
                return False
            elif choice == "1":
                self.tweets_menu()
            elif choice == "2":
                self.social_menu()
            elif choice == "3":
                self.search_menu()
            elif choice == "4":
                self.messages_menu()
            elif choice == "5":
                self.profile_menu()
            elif choice == "6":
                self.view_notifications()
                input("\n  Press Enter to continue...")
            elif choice == "7":
                self.view_trending()
                input("\n  Press Enter to continue...")
            elif choice == "8":
                self.view_explore()
                input("\n  Press Enter to continue...")
            elif choice == "9":
                self.logout()
                input("\n  Press Enter to continue...")
                return True  # Return to login screen
    
    def run(self):
        """Main run loop"""
        self.clear_screen()
        print("\n" + "="*65)
        print("  🐦 TWITTER CLONE ".center(65))
        print("  Console Edition ".center(65))
        print("="*65)
        print("\n  Loading...")
        
        import time
        time.sleep(1)
        
        while True:
            # Show login/register menu
            proceed = self.main_menu_logged_out()
            
            if not proceed:
                break
            
            # Show main menu
            go_back = self.main_menu_logged_in()
            
            if not go_back:
                break

# ============================================================
# MAIN ENTRY POINT
# ============================================================
if __name__ == "__main__":
    try:
        app = TwitterClone()
        app.run()
    except KeyboardInterrupt:
        print("\n\n  👋 Goodbye! Thanks for using Twitter Clone!\n")
    except Exception as e:
        print(f"\n  ❌ An error occurred: {e}")
        print("  Please restart the application.\n")
        import traceback
        traceback.print_exc()
