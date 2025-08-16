---
layout: chapter-post
title: "Graph Algorithms in Social Networks: A Complete Journey"
date: 2024-02-01
tags: [graphs, algorithms, social-networks, complete-guide]
author: "Anirudh Singh"
excerpt: "A comprehensive exploration of how graph algorithms evolved from competitive programming to powering modern social networks. From basic graph traversal to recommendation engines."

# Chapter Configuration
current_chapter: 1
total_chapters: 3
base_url: "/2024/02/01/graph-algorithms-social-networks"
show_toc: true

chapter_titles:
  1: "Introduction and Graph Fundamentals"
  2: "Community Detection and Clustering"
  3: "Shortest Paths and Six Degrees"

reading_time: 15
---

# Chapter 1: Introduction and Graph Fundamentals

Welcome to our comprehensive journey through graph algorithms in social networks! This multi-chapter exploration will take you from the mathematical foundations to cutting-edge applications in modern social platforms.

## What You'll Learn in This Series

This complete guide covers:

- **Chapter 1** (Current): Graph theory fundamentals and basic algorithms
- **Chapter 2**: Community detection and social clustering algorithms
- **Chapter 3**: Shortest path algorithms and the "six degrees of separation"

## The Mathematical Foundation

Social networks are essentially **graphs** where:
- **Vertices (V)**: Represent users, pages, or entities
- **Edges (E)**: Represent relationships, friendships, or interactions
- **Weights**: Can represent interaction strength, trust scores, or similarity

### Graph Representation in Memory

```cpp
// Adjacency List Representation (Most Common)
class SocialGraph {
private:
    int numUsers;
    vector<vector<int>> adjList;           // Unweighted
    vector<vector<pair<int, double>>> weightedAdjList; // Weighted
    
    // Additional metadata for social networks
    unordered_map<int, UserProfile> users;
    unordered_map<string, int> usernameToId;
    
public:
    SocialGraph(int n) : numUsers(n) {
        adjList.resize(n);
        weightedAdjList.resize(n);
    }
    
    void addFriendship(int userA, int userB, double strength = 1.0) {
        // Bidirectional friendship
        adjList[userA].push_back(userB);
        adjList[userB].push_back(userA);
        
        weightedAdjList[userA].push_back({userB, strength});
        weightedAdjList[userB].push_back({userA, strength});
    }
    
    void addFollowing(int follower, int followee, double engagement = 1.0) {
        // Directed relationship
        adjList[follower].push_back(followee);
        weightedAdjList[follower].push_back({followee, engagement});
    }
};

struct UserProfile {
    string username;
    string displayName;
    vector<string> interests;
    long long joinDate;
    int followerCount;
    int followingCount;
    double influenceScore;
};
```

## Core Graph Algorithms for Social Networks

### 1. Depth-First Search (DFS) - Finding Connected Components

In social networks, DFS helps identify isolated groups or communities:

```cpp
class SocialNetworkAnalyzer {
private:
    SocialGraph& graph;
    vector<bool> visited;
    vector<int> componentId;
    int componentCount;
    
public:
    SocialNetworkAnalyzer(SocialGraph& g) : graph(g), componentCount(0) {
        visited.resize(g.getNumUsers(), false);
        componentId.resize(g.getNumUsers(), -1);
    }
    
    void findConnectedComponents() {
        componentCount = 0;
        fill(visited.begin(), visited.end(), false);
        
        for (int user = 0; user < graph.getNumUsers(); user++) {
            if (!visited[user]) {
                dfsComponent(user, componentCount);
                componentCount++;
            }
        }
    }
    
private:
    void dfsComponent(int user, int compId) {
        visited[user] = true;
        componentId[user] = compId;
        
        for (int friend_user : graph.getAdjList()[user]) {
            if (!visited[friend_user]) {
                dfsComponent(friend_user, compId);
            }
        }
    }
    
public:
    // Analyze component sizes
    vector<int> getComponentSizes() {
        vector<int> sizes(componentCount, 0);
        for (int compId : componentId) {
            if (compId >= 0) sizes[compId]++;
        }
        return sizes;
    }
    
    // Find isolated users (components of size 1)
    vector<int> getIsolatedUsers() {
        auto sizes = getComponentSizes();
        vector<int> isolated;
        
        for (int user = 0; user < componentId.size(); user++) {
            int compId = componentId[user];
            if (compId >= 0 && sizes[compId] == 1) {
                isolated.push_back(user);
            }
        }
        return isolated;
    }
};
```

### 2. Breadth-First Search (BFS) - Social Distance

BFS helps calculate the "degrees of separation" between users:

```cpp
class SocialDistanceCalculator {
private:
    SocialGraph& graph;
    
public:
    SocialDistanceCalculator(SocialGraph& g) : graph(g) {}
    
    int calculateDistance(int userA, int userB) {
        if (userA == userB) return 0;
        
        vector<bool> visited(graph.getNumUsers(), false);
        queue<pair<int, int>> q; // {user, distance}
        
        q.push({userA, 0});
        visited[userA] = true;
        
        while (!q.empty()) {
            auto [currentUser, distance] = q.front();
            q.pop();
            
            for (int neighbor : graph.getAdjList()[currentUser]) {
                if (neighbor == userB) {
                    return distance + 1;
                }
                
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    q.push({neighbor, distance + 1});
                }
            }
        }
        
        return -1; // Not connected
    }
    
    // Calculate all distances from a source user
    vector<int> bfsAllDistances(int sourceUser) {
        vector<int> distances(graph.getNumUsers(), -1);
        queue<int> q;
        
        distances[sourceUser] = 0;
        q.push(sourceUser);
        
        while (!q.empty()) {
            int currentUser = q.front();
            q.pop();
            
            for (int neighbor : graph.getAdjList()[currentUser]) {
                if (distances[neighbor] == -1) {
                    distances[neighbor] = distances[currentUser] + 1;
                    q.push(neighbor);
                }
            }
        }
        
        return distances;
    }
    
    // Six degrees of separation analysis
    map<int, int> analyzeDegreesOfSeparation(int sourceUser) {
        auto distances = bfsAllDistances(sourceUser);
        map<int, int> degreeCount;
        
        for (int distance : distances) {
            if (distance >= 0) {
                degreeCount[distance]++;
            }
        }
        
        return degreeCount;
    }
};
```

## Real-World Application: Friend Suggestions

Let's implement a basic friend suggestion algorithm:

```cpp
class FriendSuggestionEngine {
private:
    SocialGraph& graph;
    SocialDistanceCalculator distCalc;
    
public:
    FriendSuggestionEngine(SocialGraph& g) 
        : graph(g), distCalc(g) {}
    
    struct SuggestionScore {
        int userId;
        double score;
        vector<int> mutualFriends;
        string reason;
    };
    
    vector<SuggestionScore> suggestFriends(int targetUser, int maxSuggestions = 10) {
        vector<SuggestionScore> suggestions;
        auto friends = graph.getAdjList()[targetUser];
        set<int> friendSet(friends.begin(), friends.end());
        
        // Track mutual friend counts
        unordered_map<int, vector<int>> mutualFriendsMap;
        
        // Find friends of friends (2-hop neighbors)
        for (int friendUser : friends) {
            for (int friendOfFriend : graph.getAdjList()[friendUser]) {
                if (friendOfFriend != targetUser && 
                    friendSet.find(friendOfFriend) == friendSet.end()) {
                    mutualFriendsMap[friendOfFriend].push_back(friendUser);
                }
            }
        }
        
        // Calculate suggestion scores
        for (const auto& [candidateUser, mutualList] : mutualFriendsMap) {
            double score = calculateSuggestionScore(
                targetUser, candidateUser, mutualList
            );
            
            string reason = "You have " + to_string(mutualList.size()) + 
                           " mutual friends";
            
            suggestions.push_back({
                candidateUser, score, mutualList, reason
            });
        }
        
        // Sort by score and return top suggestions
        sort(suggestions.begin(), suggestions.end(),
             [](const SuggestionScore& a, const SuggestionScore& b) {
                 return a.score > b.score;
             });
        
        if (suggestions.size() > maxSuggestions) {
            suggestions.resize(maxSuggestions);
        }
        
        return suggestions;
    }
    
private:
    double calculateSuggestionScore(int targetUser, int candidateUser,
                                   const vector<int>& mutualFriends) {
        double score = 0.0;
        
        // Base score from mutual friends
        score += mutualFriends.size() * 2.0;
        
        // Bonus for highly connected mutual friends
        for (int mutualFriend : mutualFriends) {
            int connectionCount = graph.getAdjList()[mutualFriend].size();
            score += log(connectionCount + 1) * 0.5;
        }
        
        // Consider user profiles similarity (simplified)
        auto targetProfile = graph.getUserProfile(targetUser);
        auto candidateProfile = graph.getUserProfile(candidateUser);
        
        // Interest similarity bonus
        int commonInterests = countCommonInterests(
            targetProfile.interests, candidateProfile.interests
        );
        score += commonInterests * 1.5;
        
        // Influence score consideration
        if (candidateProfile.influenceScore > 0.8) {
            score += 2.0; // Bonus for influential users
        }
        
        return score;
    }
    
    int countCommonInterests(const vector<string>& interests1,
                           const vector<string>& interests2) {
        set<string> set1(interests1.begin(), interests1.end());
        int common = 0;
        
        for (const string& interest : interests2) {
            if (set1.find(interest) != set1.end()) {
                common++;
            }
        }
        
        return common;
    }
};
```

## Performance Analysis

### Time Complexity for Social Networks

| Algorithm | Time Complexity | Space Complexity | Use Case |
|-----------|----------------|------------------|----------|
| DFS | O(V + E) | O(V) | Community detection |
| BFS | O(V + E) | O(V) | Shortest paths |
| Friend Suggestions | O(V + E) | O(V) | Recommendation |
| Connected Components | O(V + E) | O(V) | Network analysis |

Where V = number of users, E = number of connections

### Real-World Scale Considerations

For platforms like Facebook (3B users) or LinkedIn (900M users):

```cpp
// Optimizations for large-scale graphs
class ScalableSocialGraph {
private:
    // Sharded storage for horizontal scaling
    vector<unique_ptr<GraphShard>> shards;
    ConsistentHashRing hashRing;
    
    // Caching layer
    LRUCache<pair<int, int>, int> distanceCache;
    LRUCache<int, vector<int>> suggestionCache;
    
public:
    // Distributed BFS for very large graphs
    async<int> distributedBFS(int source, int target) {
        // Implementation would involve:
        // 1. Partitioning across multiple machines
        // 2. Parallel frontier expansion
        // 3. Coordination between shards
        // 4. Early termination optimizations
    }
    
    // Approximate algorithms for real-time performance
    int approximateDistance(int userA, int userB) {
        // Use landmarks and bidirectional search
        // Achieves sub-second response times
        // Even on billion-node graphs
    }
};
```

## Chapter Summary

In this foundational chapter, we've covered:

✅ **Graph representation** optimized for social networks
✅ **Core algorithms**: DFS for community detection, BFS for social distance
✅ **Practical application**: Friend suggestion algorithms
✅ **Performance considerations** for large-scale networks

### Key Takeaways

1. **Social networks are graphs** where structure reveals behavior
2. **Basic graph algorithms** form the foundation of complex social features
3. **Real-world scale** requires distributed and approximate algorithms
4. **User experience** depends on fast, relevant algorithm implementations

## What's Next?

In **Chapter 2**, we'll dive deep into **community detection algorithms**:
- Girvan-Newman algorithm for hierarchical communities
- Louvain method for modularity optimization  
- Label propagation for real-time clustering
- How Facebook and LinkedIn detect friend groups

---

*Ready to continue? Use the navigation above to proceed to Chapter 2, or explore other chapters in this comprehensive series!*

**Keywords**: graph algorithms, social networks, BFS, DFS, friend suggestions, community detection, social distance, degrees of separation
