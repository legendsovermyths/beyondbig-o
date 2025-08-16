---
layout: chapter-post
title: "Graph Algorithms in Social Networks: A Complete Journey"
date: 2024-02-01
tags: [graphs, algorithms, social-networks, complete-guide]
author: "Anirudh Singh"
excerpt: "A comprehensive exploration of how graph algorithms evolved from competitive programming to powering modern social networks. From basic graph traversal to recommendation engines."

# Chapter Configuration
current_chapter: 3
total_chapters: 3
base_url: "/2024/02/01/graph-algorithms-social-networks"
show_toc: true

chapter_title: "Shortest Paths and Six Degrees of Separation"
chapter_description: "Understanding how shortest path algorithms reveal the small-world nature of social networks"

chapter_titles:
  1: "Introduction and Graph Fundamentals"
  2: "Community Detection and Clustering"
  3: "Shortest Paths and Six Degrees"

reading_time: 16
---

# Chapter 3: Shortest Paths and Six Degrees of Separation

The concept of "six degrees of separation" suggests that everyone in the world is connected to everyone else by at most six intermediate relationships. This fascinating idea lies at the heart of many social network features and relies heavily on shortest path algorithms.

## The Small World Phenomenon

Stanley Milgram's famous 1967 experiment revealed that the average path length in social networks is surprisingly short. Modern social networks have confirmed this:

- **Facebook**: Average distance is 3.57 degrees (2016 study)
- **LinkedIn**: Average distance is 3.0 degrees for professionals  
- **Twitter**: Average distance is 4.12 degrees

## Mathematical Foundation

In a social network graph G = (V, E), the **shortest path distance** d(u,v) between users u and v is the minimum number of edges in any path connecting them.

Key metrics:
- **Diameter**: max{d(u,v)} for all pairs u,v
- **Average path length**: ∑d(u,v) / |V|²
- **Clustering coefficient**: Measures local neighborhood density

## Algorithm 1: Bidirectional BFS

For finding shortest paths between specific users, bidirectional BFS is highly efficient:

```cpp
class BidirectionalBFS {
private:
    SocialGraph& graph;
    
public:
    BidirectionalBFS(SocialGraph& g) : graph(g) {}
    
    struct PathResult {
        int distance;
        vector<int> path;
        bool found;
    };
    
    PathResult findShortestPath(int source, int target) {
        if (source == target) {
            return {0, {source}, true};
        }
        
        // Forward search from source
        queue<int> forwardQueue;
        unordered_map<int, int> forwardVisited;
        unordered_map<int, int> forwardParent;
        
        // Backward search from target
        queue<int> backwardQueue;
        unordered_map<int, int> backwardVisited;
        unordered_map<int, int> backwardParent;
        
        // Initialize
        forwardQueue.push(source);
        forwardVisited[source] = 0;
        forwardParent[source] = -1;
        
        backwardQueue.push(target);
        backwardVisited[target] = 0;
        backwardParent[target] = -1;
        
        while (!forwardQueue.empty() || !backwardQueue.empty()) {
            // Expand forward frontier
            if (!forwardQueue.empty()) {
                int meetingPoint = expandFrontier(
                    forwardQueue, forwardVisited, forwardParent,
                    backwardVisited, true
                );
                
                if (meetingPoint != -1) {
                    return constructPath(
                        source, target, meetingPoint,
                        forwardParent, backwardParent,
                        forwardVisited, backwardVisited
                    );
                }
            }
            
            // Expand backward frontier
            if (!backwardQueue.empty()) {
                int meetingPoint = expandFrontier(
                    backwardQueue, backwardVisited, backwardParent,
                    forwardVisited, false
                );
                
                if (meetingPoint != -1) {
                    return constructPath(
                        source, target, meetingPoint,
                        forwardParent, backwardParent,
                        forwardVisited, backwardVisited
                    );
                }
            }
        }
        
        return {-1, {}, false}; // No path found
    }
    
private:
    int expandFrontier(queue<int>& frontier,
                      unordered_map<int, int>& visited,
                      unordered_map<int, int>& parent,
                      const unordered_map<int, int>& otherVisited,
                      bool isForward) {
        
        int currentSize = frontier.size();
        
        for (int i = 0; i < currentSize; i++) {
            int current = frontier.front();
            frontier.pop();
            
            for (int neighbor : graph.getAdjList()[current]) {
                // Check if we've met the other search
                if (otherVisited.find(neighbor) != otherVisited.end()) {
                    return neighbor; // Meeting point found!
                }
                
                // Continue expansion if not visited
                if (visited.find(neighbor) == visited.end()) {
                    visited[neighbor] = visited[current] + 1;
                    parent[neighbor] = current;
                    frontier.push(neighbor);
                }
            }
        }
        
        return -1; // No meeting point this iteration
    }
    
    PathResult constructPath(int source, int target, int meetingPoint,
                           const unordered_map<int, int>& forwardParent,
                           const unordered_map<int, int>& backwardParent,
                           const unordered_map<int, int>& forwardVisited,
                           const unordered_map<int, int>& backwardVisited) {
        
        vector<int> path;
        
        // Build path from source to meeting point
        vector<int> forwardPath;
        int current = meetingPoint;
        while (current != -1) {
            forwardPath.push_back(current);
            auto it = forwardParent.find(current);
            current = (it != forwardParent.end()) ? it->second : -1;
        }
        reverse(forwardPath.begin(), forwardPath.end());
        
        // Build path from meeting point to target
        vector<int> backwardPath;
        current = meetingPoint;
        while (current != -1) {
            backwardPath.push_back(current);
            auto it = backwardParent.find(current);
            current = (it != backwardParent.end()) ? it->second : -1;
        }
        
        // Combine paths (avoid duplicating meeting point)
        path = forwardPath;
        path.insert(path.end(), backwardPath.begin() + 1, backwardPath.end());
        
        int distance = forwardVisited.at(meetingPoint) + 
                      backwardVisited.at(meetingPoint);
        
        return {distance, path, true};
    }
};
```

## Algorithm 2: All-Pairs Shortest Paths (Landmarks)

For computing distances to many users efficiently, we use landmark-based approximation:

```cpp
class LandmarkBasedDistances {
private:
    SocialGraph& graph;
    vector<int> landmarks;
    vector<vector<int>> landmarkDistances;
    
public:
    LandmarkBasedDistances(SocialGraph& g, int numLandmarks = 100) 
        : graph(g) {
        selectLandmarks(numLandmarks);
        precomputeDistances();
    }
    
    int approximateDistance(int source, int target) {
        int minDistance = INT_MAX;
        
        // Use triangle inequality: d(s,t) ≤ d(s,l) + d(l,t)
        for (int i = 0; i < landmarks.size(); i++) {
            int landmark = landmarks[i];
            int distanceViaLandmark = 
                landmarkDistances[i][source] + landmarkDistances[i][target];
            
            minDistance = min(minDistance, distanceViaLandmark);
        }
        
        return minDistance;
    }
    
    vector<int> getDistanceDistribution() {
        vector<int> distribution(10, 0); // Distances 0-9+
        int totalPairs = 0;
        
        for (int i = 0; i < graph.getNumUsers(); i++) {
            for (int j = i + 1; j < graph.getNumUsers(); j++) {
                int distance = approximateDistance(i, j);
                if (distance < 10) {
                    distribution[distance]++;
                } else {
                    distribution[9]++; // 9+ bucket
                }
                totalPairs++;
            }
        }
        
        return distribution;
    }
    
private:
    void selectLandmarks(int numLandmarks) {
        // Use degree-based selection for better coverage
        vector<pair<int, int>> degreeNodes;
        
        for (int i = 0; i < graph.getNumUsers(); i++) {
            int degree = graph.getAdjList()[i].size();
            degreeNodes.push_back({degree, i});
        }
        
        // Sort by degree (descending)
        sort(degreeNodes.rbegin(), degreeNodes.rend());
        
        // Select top-k high-degree nodes
        for (int i = 0; i < min(numLandmarks, (int)degreeNodes.size()); i++) {
            landmarks.push_back(degreeNodes[i].second);
        }
        
        // Add random nodes for diversity
        random_device rd;
        mt19937 gen(rd());
        uniform_int_distribution<> dis(0, graph.getNumUsers() - 1);
        
        set<int> landmarkSet(landmarks.begin(), landmarks.end());
        while (landmarks.size() < numLandmarks && 
               landmarks.size() < graph.getNumUsers()) {
            int randomNode = dis(gen);
            if (landmarkSet.find(randomNode) == landmarkSet.end()) {
                landmarks.push_back(randomNode);
                landmarkSet.insert(randomNode);
            }
        }
    }
    
    void precomputeDistances() {
        landmarkDistances.resize(landmarks.size());
        
        for (int i = 0; i < landmarks.size(); i++) {
            landmarkDistances[i] = computeBFS(landmarks[i]);
        }
    }
    
    vector<int> computeBFS(int source) {
        vector<int> distances(graph.getNumUsers(), -1);
        queue<int> q;
        
        distances[source] = 0;
        q.push(source);
        
        while (!q.empty()) {
            int current = q.front();
            q.pop();
            
            for (int neighbor : graph.getAdjList()[current]) {
                if (distances[neighbor] == -1) {
                    distances[neighbor] = distances[current] + 1;
                    q.push(neighbor);
                }
            }
        }
        
        return distances;
    }
};
```

## Social Distance Analytics

Let's implement comprehensive social distance analysis:

```cpp
class SocialDistanceAnalyzer {
private:
    SocialGraph& graph;
    LandmarkBasedDistances& landmarks;
    
public:
    SocialDistanceAnalyzer(SocialGraph& g, LandmarkBasedDistances& l) 
        : graph(g), landmarks(l) {}
    
    struct NetworkMetrics {
        double averagePathLength;
        int diameter;
        double clusteringCoefficient;
        vector<int> degreeDistribution;
        vector<int> distanceDistribution;
        double smallWorldIndex;
    };
    
    NetworkMetrics analyzeNetwork() {
        NetworkMetrics metrics;
        
        // Calculate average path length and diameter
        auto [avgPath, diameter] = computePathStatistics();
        metrics.averagePathLength = avgPath;
        metrics.diameter = diameter;
        
        // Calculate clustering coefficient
        metrics.clusteringCoefficient = computeClusteringCoefficient();
        
        // Get distributions
        metrics.degreeDistribution = computeDegreeDistribution();
        metrics.distanceDistribution = landmarks.getDistanceDistribution();
        
        // Calculate small-world index
        metrics.smallWorldIndex = computeSmallWorldIndex(
            metrics.averagePathLength, metrics.clusteringCoefficient
        );
        
        return metrics;
    }
    
    vector<int> findInfluentialBridges() {
        vector<int> bridges;
        
        // Find nodes with high betweenness centrality
        auto betweenness = computeBetweennessCentrality();
        
        // Sort by betweenness score
        vector<pair<double, int>> scoredNodes;
        for (int i = 0; i < betweenness.size(); i++) {
            scoredNodes.push_back({betweenness[i], i});
        }
        sort(scoredNodes.rbegin(), scoredNodes.rend());
        
        // Return top 10% as influential bridges
        int numBridges = max(1, (int)(scoredNodes.size() * 0.1));
        for (int i = 0; i < numBridges; i++) {
            bridges.push_back(scoredNodes[i].second);
        }
        
        return bridges;
    }
    
private:
    pair<double, int> computePathStatistics() {
        double totalDistance = 0.0;
        int maxDistance = 0;
        int validPairs = 0;
        
        // Sample pairs to estimate (full computation too expensive)
        random_device rd;
        mt19937 gen(rd());
        uniform_int_distribution<> dis(0, graph.getNumUsers() - 1);
        
        int sampleSize = min(10000, graph.getNumUsers() * graph.getNumUsers());
        
        for (int sample = 0; sample < sampleSize; sample++) {
            int source = dis(gen);
            int target = dis(gen);
            
            if (source != target) {
                int distance = landmarks.approximateDistance(source, target);
                if (distance > 0) {
                    totalDistance += distance;
                    maxDistance = max(maxDistance, distance);
                    validPairs++;
                }
            }
        }
        
        double avgDistance = validPairs > 0 ? totalDistance / validPairs : 0.0;
        return {avgDistance, maxDistance};
    }
    
    double computeClusteringCoefficient() {
        double totalCC = 0.0;
        int validNodes = 0;
        
        for (int node = 0; node < graph.getNumUsers(); node++) {
            auto neighbors = graph.getAdjList()[node];
            if (neighbors.size() < 2) continue;
            
            int possibleTriangles = neighbors.size() * (neighbors.size() - 1) / 2;
            int actualTriangles = 0;
            
            // Count triangles
            for (int i = 0; i < neighbors.size(); i++) {
                for (int j = i + 1; j < neighbors.size(); j++) {
                    if (graph.areConnected(neighbors[i], neighbors[j])) {
                        actualTriangles++;
                    }
                }
            }
            
            totalCC += (double)actualTriangles / possibleTriangles;
            validNodes++;
        }
        
        return validNodes > 0 ? totalCC / validNodes : 0.0;
    }
    
    vector<double> computeBetweennessCentrality() {
        vector<double> betweenness(graph.getNumUsers(), 0.0);
        
        // Use sampling for efficiency on large graphs
        vector<int> sampleNodes;
        if (graph.getNumUsers() > 1000) {
            // Sample 100 nodes
            random_device rd;
            mt19937 gen(rd());
            uniform_int_distribution<> dis(0, graph.getNumUsers() - 1);
            
            set<int> sampledSet;
            while (sampledSet.size() < 100) {
                sampledSet.insert(dis(gen));
            }
            sampleNodes.assign(sampledSet.begin(), sampledSet.end());
        } else {
            // Use all nodes for small graphs
            for (int i = 0; i < graph.getNumUsers(); i++) {
                sampleNodes.push_back(i);
            }
        }
        
        for (int source : sampleNodes) {
            // Single-source shortest paths with path counting
            auto pathData = computeShortestPathsWithCounting(source);
            
            // Accumulate betweenness scores
            for (int target = 0; target < graph.getNumUsers(); target++) {
                if (source != target) {
                    updateBetweennessFromPaths(
                        betweenness, source, target, pathData
                    );
                }
            }
        }
        
        // Normalize by sampling factor
        double normalizationFactor = (double)graph.getNumUsers() / sampleNodes.size();
        for (double& score : betweenness) {
            score *= normalizationFactor;
        }
        
        return betweenness;
    }
};
```

## Real-World Applications

### LinkedIn's "People You May Know"

LinkedIn uses shortest path insights to suggest connections:

```cpp
class LinkedInPeopleYouMayKnow {
    struct ConnectionSuggestion {
        int userId;
        int distance;
        vector<int> shortestPath;
        double relevanceScore;
        string reason;
    };
    
    vector<ConnectionSuggestion> generateSuggestions(int targetUser) {
        vector<ConnectionSuggestion> suggestions;
        
        // Find users at distance 2-3 (not direct connections)
        for (int candidate = 0; candidate < graph.getNumUsers(); candidate++) {
            if (candidate == targetUser) continue;
            
            int distance = landmarks.approximateDistance(targetUser, candidate);
            
            if (distance >= 2 && distance <= 3) {
                auto pathResult = bidirectionalBFS.findShortestPath(
                    targetUser, candidate
                );
                
                if (pathResult.found) {
                    double score = calculateRelevanceScore(
                        targetUser, candidate, pathResult.path
                    );
                    
                    string reason = buildReasonString(pathResult.path);
                    
                    suggestions.push_back({
                        candidate, distance, pathResult.path, score, reason
                    });
                }
            }
        }
        
        // Sort by relevance and return top suggestions
        sort(suggestions.begin(), suggestions.end(),
             [](const auto& a, const auto& b) {
                 return a.relevanceScore > b.relevanceScore;
             });
        
        return suggestions;
    }
    
private:
    double calculateRelevanceScore(int user, int candidate,
                                 const vector<int>& path) {
        double score = 0.0;
        
        // Shorter paths get higher scores
        score += 10.0 / path.size();
        
        // Bonus for high-degree intermediaries (influencers)
        for (int i = 1; i < path.size() - 1; i++) {
            int intermediary = path[i];
            int degree = graph.getAdjList()[intermediary].size();
            score += log(degree + 1) * 0.5;
        }
        
        // Professional similarity bonus
        auto userProfile = graph.getUserProfile(user);
        auto candidateProfile = graph.getUserProfile(candidate);
        
        if (userProfile.industry == candidateProfile.industry) {
            score += 5.0;
        }
        
        if (userProfile.company == candidateProfile.company) {
            score += 8.0;
        }
        
        return score;
    }
};
```

### Facebook's Six Degrees Feature

Facebook actually built a tool to show the path between any two users:

```cpp
class FacebookSixDegrees {
    struct DegreesResult {
        bool pathExists;
        int degrees;
        vector<string> pathDescription;
        vector<int> userPath;
        double confidence;
    };
    
    DegreesResult findConnectionPath(int userA, int userB) {
        // Use privacy-respecting path finding
        auto pathResult = findPrivateAwarePath(userA, userB);
        
        if (!pathResult.found) {
            return {false, -1, {}, {}, 0.0};
        }
        
        // Build human-readable description
        vector<string> descriptions;
        for (int i = 0; i < pathResult.path.size() - 1; i++) {
            int current = pathResult.path[i];
            int next = pathResult.path[i + 1];
            
            string relationship = inferRelationshipType(current, next);
            descriptions.push_back(relationship);
        }
        
        double confidence = calculatePathConfidence(pathResult.path);
        
        return {
            true,
            pathResult.distance,
            descriptions,
            pathResult.path,
            confidence
        };
    }
    
private:
    BidirectionalBFS::PathResult findPrivateAwarePath(int userA, int userB) {
        // Only use paths through users with appropriate privacy settings
        // This is a simplified version - real implementation is more complex
        
        BidirectionalBFS bfs(graph);
        return bfs.findShortestPath(userA, userB);
    }
    
    string inferRelationshipType(int userA, int userB) {
        // Analyze interaction patterns to infer relationship
        auto interactionData = graph.getInteractionData(userA, userB);
        
        if (interactionData.familyIndicators > 0.8) {
            return "family member of";
        } else if (interactionData.workIndicators > 0.7) {
            return "colleague of";
        } else if (interactionData.schoolIndicators > 0.7) {
            return "went to school with";
        } else {
            return "friends with";
        }
    }
};
```

## Performance Analysis

### Computational Complexity

| Algorithm | Time Complexity | Space Complexity | Use Case |
|-----------|----------------|------------------|----------|
| Bidirectional BFS | O(b^(d/2)) | O(b^(d/2)) | Single path |
| Landmark-based | O(k × (V + E)) | O(k × V) | Many queries |
| All-pairs BFS | O(V × (V + E)) | O(V²) | Complete analysis |

Where:
- b = branching factor (average degree)
- d = shortest path distance
- k = number of landmarks

### Real-World Performance

For Facebook-scale networks (3B users, 200B edges):

```cpp
class ScalableShortestPaths {
    // Production optimizations:
    
    // 1. Geographic partitioning
    // - Users in same region likely have shorter paths
    // - Reduces cross-datacenter queries
    
    // 2. Hierarchical landmarks
    // - Country-level, region-level, city-level landmarks
    // - Multi-resolution distance estimation
    
    // 3. Caching strategies
    // - Cache frequent path queries
    // - Precompute celebrity distances
    
    // 4. Approximation algorithms
    // - Trade accuracy for speed
    // - Real-time response requirements
};
```

## Chapter Summary

We've explored how shortest path algorithms reveal the small-world structure of social networks:

✅ **Bidirectional BFS** for efficient path finding between specific users
✅ **Landmark-based approximation** for scalable distance estimation
✅ **Social distance analytics** for understanding network structure
✅ **Real-world applications** in connection suggestions and network analysis

### Key Insights

1. **Social networks are small worlds** - surprisingly short average path lengths
2. **Bidirectional search** dramatically improves path-finding efficiency
3. **Approximation algorithms** enable real-time queries on massive networks
4. **Bridge nodes** with high betweenness centrality are crucial for connectivity

## What's Next?

In **Chapter 4**, we'll explore **influence and information propagation**:
- Viral content algorithms
- Influence maximization problems
- Information cascade models
- How social platforms optimize content spread

---

*The journey continues with fascinating algorithms that govern how information flows through social networks!*
