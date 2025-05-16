Yeah I only want to load and cache users in a given channel I am accessing. I also want to be mindful of the request rate going in to not trip any alarms. But to start, just user IDs is fine and no user detail loading. I just want to get messages working first

--------------------------------------------------------------------------------

	*Augment*

That's a good approach. Here's how to implement it:

1. First, disable the global user cache as we discussed using the `SLACK_MCP_DISABLE_USERS_CACHE` environment variable.

2. For the initial implementation, just display user IDs instead of names when showing channel messages. This will work without any additional API calls.

3. Later, you can implement a per-channel user cache that only loads users for channels you're actively viewing:

````go path=pkg/provider/api.go mode=EDIT
// Add this to your ApiProvider struct
type ApiProvider struct {
    // existing fields...
    channelUsers     map[string]map[string]slack.User // map[channelID]map[userID]User
    channelUsersMutex sync.RWMutex
}

// Add this method to fetch users for a specific channel
func (ap *ApiProvider) GetUserForChannel(channelID, userID string) (slack.User, error) {
    // Check if we already have this user cached for this channel
    ap.channelUsersMutex.RLock()
    if channelCache, ok := ap.channelUsers[channelID]; ok {
        if user, found := channelCache[userID]; found {
            ap.channelUsersMutex.RUnlock()
            return user, nil
        }
    }
    ap.channelUsersMutex.RUnlock()
    
    // User not in cache, fetch from API
    user, err := ap.Client().GetUserInfo(userID)
    if err != nil {
        return slack.User{}, err
    }
    
    // Add to channel cache
    ap.channelUsersMutex.Lock()
    if ap.channelUsers == nil {
        ap.channelUsers = make(map[string]map[string]slack.User)
    }
    if ap.channelUsers[channelID] == nil {
        ap.channelUsers[channelID] = make(map[string]slack.User)
    }
    ap.channelUsers[channelID][userID] = *user
    ap.channelUsersMutex.Unlock()
    
    // Add rate limiting if needed
    time.Sleep(100 * time.Millisecond) // Simple rate limiting
    
    return *user, nil
}
````

This implementation:

1. Starts with just user IDs in messages (no additional API calls)
2. When you're ready to add user details, it:
   - Only caches users for channels you're actively viewing
   - Fetches user details on-demand
   - Includes basic rate limiting to avoid triggering Slack's rate limits
   - Uses a per-channel cache structure that's much more memory efficient

For your initial goal of "just getting messages working first with user IDs," simply disabling the global cache via the environment variable is sufficient. The per-channel caching can be implemented later when you're ready to enhance the user experience.

