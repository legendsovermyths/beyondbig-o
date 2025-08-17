---
layout: post
title: "The Complete Guide to Writing for Beyond Big-O"
date: 2024-03-01
author: "Anirudh Singh"
excerpt: "A comprehensive guide covering everything from basic blog formatting to interactive visualizations with p5.js and D3.js. Learn how to create engaging algorithm content that bridges theory and practice."
tags: ["guide", "writing", "markdown", "interactive", "p5js", "d3js", "mathjax"]
reading_time: 20
---

Welcome to the complete guide for writing content on Beyond Big-O! This guide will walk you through everything you need to know to create engaging, interactive algorithm explanations that truly embody our "from contest to codebase" philosophy.

## 📝 Table of Contents

1. [Blog Post Basics](#blog-post-basics)
2. [Regular Blog Posts](#regular-blog-posts)
3. [Multi-Chapter Blog Posts](#multi-chapter-blog-posts)
4. [Mathematical Content with MathJax](#mathematical-content-with-mathjax)
5. [Interactive Visualizations with p5.js](#interactive-visualizations-with-p5js)
6. [Advanced Visualizations with D3.js](#advanced-visualizations-with-d3js)
7. [Code Highlighting and Examples](#code-highlighting-and-examples)
8. [Best Practices and Style Guide](#best-practices-and-style-guide)

---

## Blog Post Basics

### File Naming Convention

All blog posts must follow this exact format:
```
_posts/YYYY-MM-DD-title-with-hyphens.md
```

**Examples:**
- ✅ `2024-03-15-quicksort-optimization-techniques.md`
- ✅ `2024-04-01-dynamic-programming-memoization.md`
- ❌ `quicksort.md` (missing date)
- ❌ `2024-3-15-post.md` (incorrect date format)

### Directory Structure
```
_posts/
├── 2024-01-15-dijkstra-to-microservices.md
├── 2024-02-01-graph-algorithms-chapter-1.md
├── 2024-02-01-graph-algorithms-chapter-2.md
└── 2024-03-01-your-new-post.md
```

---

## Regular Blog Posts

### Basic Template

Create a new file in `_posts/` with this template:

```yaml
---
layout: post
title: "Your Algorithm Post Title"
date: 2024-03-15
author: "Anirudh Singh"
excerpt: "A compelling description that appears in previews and social media shares. Keep it under 160 characters for optimal SEO."
tags: ["algorithms", "data-structures", "optimization"]
reading_time: 12
---

Your content starts here...
```

### Frontmatter Explained

| Field | Required | Description |
|-------|----------|-------------|
| `layout` | ✅ | Always use `post` for regular articles |
| `title` | ✅ | The main title (appears in navigation, SEO) |
| `date` | ✅ | Publication date (YYYY-MM-DD format) |
| `author` | ✅ | Your name |
| `excerpt` | ✅ | Brief description for previews |
| `tags` | ✅ | Array of relevant tags |
| `reading_time` | ⚠️ | Estimated reading time in minutes |

### Content Structure

```markdown
## Introduction
Brief hook that connects competitive programming to real-world applications.

## The Algorithm
Core algorithmic explanation with complexity analysis.

## Mathematical Foundation
Rigorous mathematical treatment (see MathJax section below).

## Implementation
Clean, production-ready code examples.

## Real-World Applications
How this algorithm is used in production systems.

## Interactive Demo
(Optional) Live visualization or interactive example.

## Conclusion
Key takeaways and connections to broader software engineering principles.
```

---

## Multi-Chapter Blog Posts

For complex topics that require multiple parts, use the chapter system.

### Chapter Template

**File naming:** `YYYY-MM-DD-series-title-chapter-N.md`

```yaml
---
layout: chapter-post
title: "Graph Algorithms in Social Networks: A Complete Journey"
date: 2024-02-01
author: "Anirudh Singh"
excerpt: "A comprehensive exploration of graph algorithms through the lens of social network analysis."
tags: ["graphs", "social-networks", "algorithms"]
reading_time: 15

# Chapter Configuration
current_chapter: 1
total_chapters: 3
base_url: "/2024/02/01/graph-algorithms-social-networks"
show_toc: true

chapter_titles:
  1: "Introduction and Graph Fundamentals"
  2: "Community Detection and Clustering"
  3: "Shortest Paths and Six Degrees"

# Chapter-specific
chapter_title: "Introduction and Graph Fundamentals"
chapter_description: "Building the mathematical foundation for understanding social networks as graphs."
---

Your chapter content here...
```

### Chapter Configuration Explained

| Field | Description |
|-------|-------------|
| `layout` | Must be `chapter-post` |
| `current_chapter` | Which chapter this file represents (1, 2, 3...) |
| `total_chapters` | Total number of chapters in the series |
| `base_url` | Common URL prefix for all chapters |
| `chapter_titles` | Map of chapter numbers to titles |
| `chapter_title` | Title specific to this chapter |
| `chapter_description` | Optional description for this chapter |

### Creating a Chapter Series

1. **Plan your chapters** - Decide on total number and titles
2. **Create each file** with consistent `base_url` and `total_chapters`
3. **Update `chapter_titles`** map in all files when adding/removing chapters
4. **Use consistent tags** across all chapters

**Example series:**
```
_posts/2024-04-01-dynamic-programming-mastery-chapter-1.md
_posts/2024-04-01-dynamic-programming-mastery-chapter-2.md
_posts/2024-04-01-dynamic-programming-mastery-chapter-3.md
```

---

## Mathematical Content with MathJax

Our blog supports full LaTeX math rendering through MathJax.

### Inline Math
Use single dollar signs for inline equations:
```markdown
The time complexity is $O(n \log n)$ for the average case.
Binary search achieves $O(\log n)$ by halving the search space.
```

**Renders as:** The time complexity is $O(n \log n)$ for the average case.

### Block Math
Use double dollar signs for centered equations:
```markdown
$$
T(n) = \begin{cases}
1 & \text{if } n = 1 \\
2T(n/2) + n & \text{if } n > 1
\end{cases}
$$
```

**Renders as:**
$$
T(n) = \begin{cases}
1 & \text{if } n = 1 \\
2T(n/2) + n & \text{if } n > 1
\end{cases}
$$

### Advanced Math Examples

**Matrices:**
```latex
$$
\begin{bmatrix}
a_{11} & a_{12} & \cdots & a_{1n} \\
a_{21} & a_{22} & \cdots & a_{2n} \\
\vdots & \vdots & \ddots & \vdots \\
a_{m1} & a_{m2} & \cdots & a_{mn}
\end{bmatrix}
$$
```

**Summations and Products:**
```latex
$$
\sum_{i=1}^{n} i^2 = \frac{n(n+1)(2n+1)}{6}
$$

$$
\prod_{i=1}^{n} i = n!
$$
```

**Algorithms in Math:**
```latex
$$
\text{QuickSort}(A, p, r) = \begin{cases}
\text{Partition}(A, p, r) & \text{if } p < r \\
\text{QuickSort}(A, p, q-1) & \\
\text{QuickSort}(A, q+1, r) &
\end{cases}
$$
```

### Math Best Practices

1. **Use descriptive variable names** in equations
2. **Explain notation** before using it
3. **Break complex equations** into steps
4. **Connect math to code** - show the relationship

---

## Interactive Visualizations with p5.js

p5.js is perfect for algorithm animations and interactive demonstrations.

### Basic p5.js Setup

```html
<div id="p5-container" style="display: flex; justify-content: center; margin: 2rem 0;"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js"></script>
<script>
let sketch = function(p) {
    let array = [];
    let sorting = false;
    
    p.setup = function() {
        let canvas = p.createCanvas(600, 400);
        canvas.parent('p5-container');
        
        // Initialize random array
        for (let i = 0; i < 20; i++) {
            array.push(p.random(10, 300));
        }
    };
    
    p.draw = function() {
        // Use CSS variables for theming
        let bgColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--bg-primary').trim();
        let accentColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--accent-primary').trim();
            
        p.background(bgColor);
        
        // Draw array bars
        let barWidth = p.width / array.length;
        for (let i = 0; i < array.length; i++) {
            p.fill(accentColor);
            p.rect(i * barWidth, p.height - array[i], barWidth - 2, array[i]);
        }
    };
    
    p.mousePressed = function() {
        if (!sorting) {
            // Shuffle array
            array = p.shuffle(array);
        }
    };
};

new p5(sketch);
</script>
```

### Advanced p5.js Example: Bubble Sort Animation

```html
<div class="algorithm-demo">
    <div id="bubble-sort-demo"></div>
    <div class="demo-controls">
        <button id="start-sort">Start Sorting</button>
        <button id="reset-array">Reset Array</button>
        <span id="sort-status">Click start to begin</span>
    </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js"></script>
<script>
let bubbleSortSketch = function(p) {
    let array = [];
    let i = 0, j = 0;
    let sorting = false;
    let completed = false;
    
    p.setup = function() {
        let canvas = p.createCanvas(700, 300);
        canvas.parent('bubble-sort-demo');
        resetArray();
    };
    
    function resetArray() {
        array = [];
        for (let k = 0; k < 30; k++) {
            array.push({
                value: p.random(10, 250),
                color: 'default',
                id: k
            });
        }
        i = 0;
        j = 0;
        sorting = false;
        completed = false;
    }
    
    p.draw = function() {
        // Theme-aware colors
        let bgColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--bg-secondary').trim();
        let defaultColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--accent-primary').trim();
        let compareColor = '#ef4444';
        let swapColor = '#22c55e';
        
        p.background(bgColor);
        
        let barWidth = p.width / array.length;
        
        for (let k = 0; k < array.length; k++) {
            // Set color based on state
            switch(array[k].color) {
                case 'comparing':
                    p.fill(compareColor);
                    break;
                case 'swapping':
                    p.fill(swapColor);
                    break;
                case 'sorted':
                    p.fill('#10b981');
                    break;
                default:
                    p.fill(defaultColor);
            }
            
            p.rect(k * barWidth, p.height - array[k].value, barWidth - 2, array[k].value);
            
            // Reset color for next frame
            if (array[k].color !== 'sorted') {
                array[k].color = 'default';
            }
        }
        
        // Perform one step of bubble sort
        if (sorting && !completed) {
            bubbleSortStep();
        }
    };
    
    function bubbleSortStep() {
        if (i < array.length - 1) {
            if (j < array.length - i - 1) {
                // Mark elements being compared
                array[j].color = 'comparing';
                array[j + 1].color = 'comparing';
                
                if (array[j].value > array[j + 1].value) {
                    // Swap elements
                    [array[j], array[j + 1]] = [array[j + 1], array[j]];
                    array[j].color = 'swapping';
                    array[j + 1].color = 'swapping';
                }
                j++;
            } else {
                // Mark last element as sorted
                array[array.length - 1 - i].color = 'sorted';
                j = 0;
                i++;
            }
        } else {
            // Sorting complete
            array[0].color = 'sorted';
            completed = true;
            sorting = false;
            document.getElementById('sort-status').textContent = 'Sorting completed!';
        }
    }
    
    // Global functions for controls
    window.startBubbleSort = function() {
        if (!sorting && !completed) {
            sorting = true;
            document.getElementById('sort-status').textContent = 'Sorting in progress...';
        }
    };
    
    window.resetBubbleSort = function() {
        resetArray();
        document.getElementById('sort-status').textContent = 'Click start to begin';
    };
};

new p5(bubbleSortSketch);

// Event listeners
document.getElementById('start-sort').addEventListener('click', window.startBubbleSort);
document.getElementById('reset-array').addEventListener('click', window.resetBubbleSort);
</script>

<style>
.algorithm-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-color);
}

.demo-controls {
    display: flex;
    gap: 1rem;
    align-items: center;
    justify-content: center;
    margin-top: 1rem;
    flex-wrap: wrap;
}

.demo-controls button {
    padding: 0.75rem 1.5rem;
    background: var(--accent-primary);
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 500;
    transition: background 0.2s ease;
}

.demo-controls button:hover {
    background: var(--accent-secondary);
}
</style>
```

### p5.js Best Practices

1. **Use theme colors** - Access CSS variables for consistent theming
2. **Responsive sizing** - Make canvases adapt to screen size
3. **Performance** - Limit frame rate for complex animations
4. **Accessibility** - Provide controls and descriptions

```javascript
// Theme-aware color function
function getThemeColor(variable) {
    return getComputedStyle(document.documentElement)
        .getPropertyValue(variable).trim();
}

// Responsive canvas
p.setup = function() {
    let container = document.getElementById('container');
    let width = Math.min(container.offsetWidth, 800);
    let height = width * 0.6; // Maintain aspect ratio
    let canvas = p.createCanvas(width, height);
    canvas.parent('container');
};
```

---

## Advanced Visualizations with D3.js

D3.js is ideal for complex data visualizations and graph algorithms.

### Basic D3.js Setup

```html
<div id="d3-visualization" style="margin: 2rem 0;"></div>

<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', function() {
    // Get theme colors
    const getThemeColor = (variable) => {
        return getComputedStyle(document.documentElement)
            .getPropertyValue(variable).trim();
    };
    
    const width = 600;
    const height = 400;
    
    const svg = d3.select("#d3-visualization")
        .append("svg")
        .attr("width", width)
        .attr("height", height)
        .style("background", getThemeColor('--bg-secondary'))
        .style("border-radius", "8px")
        .style("border", `1px solid ${getThemeColor('--border-color')}`);
    
    // Your D3 visualization code here
});
</script>
```

### Advanced D3.js Example: Graph Traversal

```html
<div class="graph-demo">
    <div id="graph-visualization"></div>
    <div class="graph-controls">
        <button id="start-bfs">Start BFS</button>
        <button id="start-dfs">Start DFS</button>
        <button id="reset-graph">Reset</button>
        <select id="start-node">
            <option value="0">Start from Node 0</option>
            <option value="1">Start from Node 1</option>
            <option value="2">Start from Node 2</option>
        </select>
    </div>
</div>

<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', function() {
    const width = 700;
    const height = 400;
    
    // Sample graph data
    const nodes = [
        {id: 0, x: 150, y: 200},
        {id: 1, x: 300, y: 100},
        {id: 2, x: 450, y: 200},
        {id: 3, x: 300, y: 300},
        {id: 4, x: 550, y: 150}
    ];
    
    const links = [
        {source: 0, target: 1},
        {source: 1, target: 2},
        {source: 0, target: 3},
        {source: 2, target: 4},
        {source: 1, target: 3}
    ];
    
    const svg = d3.select("#graph-visualization")
        .append("svg")
        .attr("width", width)
        .attr("height", height)
        .style("background", getComputedStyle(document.documentElement)
            .getPropertyValue('--bg-primary').trim())
        .style("border-radius", "8px")
        .style("border", `1px solid ${getComputedStyle(document.documentElement)
            .getPropertyValue('--border-color').trim()}`);
    
    // Draw edges
    const edges = svg.selectAll("line")
        .data(links)
        .enter()
        .append("line")
        .attr("x1", d => nodes[d.source].x)
        .attr("y1", d => nodes[d.source].y)
        .attr("x2", d => nodes[d.target].x)
        .attr("y2", d => nodes[d.target].y)
        .attr("stroke", getComputedStyle(document.documentElement)
            .getPropertyValue('--border-color').trim())
        .attr("stroke-width", 2);
    
    // Draw nodes
    const nodeElements = svg.selectAll("circle")
        .data(nodes)
        .enter()
        .append("circle")
        .attr("cx", d => d.x)
        .attr("cy", d => d.y)
        .attr("r", 20)
        .attr("fill", getComputedStyle(document.documentElement)
            .getPropertyValue('--accent-primary').trim())
        .attr("stroke", "white")
        .attr("stroke-width", 2);
    
    // Add node labels
    svg.selectAll("text")
        .data(nodes)
        .enter()
        .append("text")
        .attr("x", d => d.x)
        .attr("y", d => d.y + 5)
        .attr("text-anchor", "middle")
        .attr("fill", "white")
        .attr("font-weight", "bold")
        .text(d => d.id);
    
    // BFS Animation
    async function animateBFS(startNode) {
        const visited = new Set();
        const queue = [startNode];
        
        while (queue.length > 0) {
            const current = queue.shift();
            
            if (visited.has(current)) continue;
            
            // Animate current node
            nodeElements.filter(d => d.id === current)
                .transition()
                .duration(500)
                .attr("fill", "#22c55e");
            
            visited.add(current);
            
            // Add neighbors to queue
            links.forEach(link => {
                const neighbor = link.source === current ? link.target : 
                               link.target === current ? link.source : null;
                
                if (neighbor !== null && !visited.has(neighbor)) {
                    queue.push(neighbor);
                }
            });
            
            // Wait for animation
            await new Promise(resolve => setTimeout(resolve, 800));
        }
    }
    
    // Event listeners
    document.getElementById('start-bfs').addEventListener('click', () => {
        const startNode = parseInt(document.getElementById('start-node').value);
        resetGraph();
        animateBFS(startNode);
    });
    
    function resetGraph() {
        nodeElements.attr("fill", getComputedStyle(document.documentElement)
            .getPropertyValue('--accent-primary').trim());
    }
    
    document.getElementById('reset-graph').addEventListener('click', resetGraph);
});
</script>

<style>
.graph-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-color);
}

.graph-controls {
    display: flex;
    gap: 1rem;
    align-items: center;
    justify-content: center;
    margin-top: 1rem;
    flex-wrap: wrap;
}

.graph-controls button, .graph-controls select {
    padding: 0.5rem 1rem;
    background: var(--accent-primary);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-weight: 500;
}

.graph-controls select {
    background: var(--bg-primary);
    color: var(--text-primary);
    border: 1px solid var(--border-color);
}
</style>
```

### D3.js Best Practices

1. **Responsive design** - Use viewBox for scalable SVGs
2. **Smooth transitions** - Use D3's transition API
3. **Theme integration** - Access CSS variables for colors
4. **Data binding** - Leverage D3's data-driven approach
5. **Performance** - Optimize for large datasets

```javascript
// Responsive SVG
const svg = d3.select("#container")
    .append("svg")
    .attr("viewBox", `0 0 ${width} ${height}`)
    .attr("preserveAspectRatio", "xMidYMid meet")
    .style("width", "100%")
    .style("height", "auto");

// Theme-aware styling
const updateTheme = () => {
    const isDark = document.documentElement.getAttribute('data-theme') === 'dark';
    svg.style("background", isDark ? "#1a1a1a" : "#ffffff");
};

// Listen for theme changes
document.addEventListener('themeChanged', updateTheme);
```

---

## Code Highlighting and Examples

Our blog uses Prism.js with a custom Mocha theme for syntax highlighting.

### Supported Languages

```cpp
// C++ - Perfect for competitive programming
#include <vector>
#include <algorithm>

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> map;
        for (int i = 0; i < nums.size(); i++) {
            int complement = target - nums[i];
            if (map.find(complement) != map.end()) {
                return {map[complement], i};
            }
            map[nums[i]] = i;
        }
        return {};
    }
};
```

```python
# Python - Great for prototyping and explanation
def binary_search(arr, target):
    """
    Performs binary search on a sorted array.
    
    Time Complexity: O(log n)
    Space Complexity: O(1)
    """
    left, right = 0, len(arr) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return -1
```

```javascript
// JavaScript - For web applications and visualizations
class SegmentTree {
    constructor(arr) {
        this.n = arr.length;
        this.tree = new Array(4 * this.n);
        this.build(arr, 1, 0, this.n - 1);
    }
    
    build(arr, node, start, end) {
        if (start === end) {
            this.tree[node] = arr[start];
        } else {
            const mid = Math.floor((start + end) / 2);
            this.build(arr, 2 * node, start, mid);
            this.build(arr, 2 * node + 1, mid + 1, end);
            this.tree[node] = this.tree[2 * node] + this.tree[2 * node + 1];
        }
    }
}
```

```sql
-- SQL - For database-related algorithm applications
CREATE INDEX idx_user_score ON users(score);

-- This query benefits from our index
SELECT user_id, username, score 
FROM users 
WHERE score > 1000 
ORDER BY score DESC 
LIMIT 10;
```

### Code Best Practices

1. **Include comments** explaining key concepts
2. **Use descriptive variable names**
3. **Add complexity analysis** in comments
4. **Show multiple language implementations** when relevant
5. **Include error handling** for production code

---

## Best Practices and Style Guide

### Writing Style

#### Voice and Tone
- **Conversational yet authoritative** - "Let's explore how..."
- **Bridge theory and practice** - Always connect to real-world applications
- **Encourage experimentation** - "Try modifying the parameters..."

#### Structure
1. **Hook** - Start with an interesting problem or observation
2. **Theory** - Explain the algorithmic concept
3. **Mathematics** - Provide rigorous analysis
4. **Implementation** - Show clean, production-ready code
5. **Applications** - Connect to real-world systems
6. **Interactive elements** - Let readers experiment
7. **Conclusion** - Tie everything together

### Content Guidelines

#### Algorithm Posts Should Include:
- [ ] **Time and space complexity** analysis
- [ ] **Mathematical proof** or intuition
- [ ] **Multiple implementations** (competitive programming + production)
- [ ] **Real-world applications** 
- [ ] **Common pitfalls** and edge cases
- [ ] **Interactive visualization** (when possible)

#### Real-World Connections
Always answer: "Where is this used in production?"

**Examples:**
- Binary search → Database indexing, Git bisect
- Graph algorithms → Social networks, recommendation systems
- Dynamic programming → Resource allocation, optimization
- Segment trees → Range queries, database systems

### SEO and Discoverability

#### Title Best Practices
- **Include algorithm name** for searchability
- **Mention real-world application** 
- **Keep under 60 characters**

**Good titles:**
- "Binary Search: From Theory to Interactive Reality"
- "Dijkstra's Algorithm: From Contest to Microservices"
- "Segment Trees: Powering Database Range Queries"

#### Excerpt Guidelines
- **Under 160 characters** for optimal SEO
- **Include key algorithms** mentioned
- **Highlight unique value** (interactive, real-world focus)

#### Tag Strategy
Use 3-5 relevant tags:
- **Algorithm names**: `binary-search`, `dijkstra`, `dynamic-programming`
- **Categories**: `algorithms`, `data-structures`, `optimization`
- **Applications**: `databases`, `networking`, `systems`
- **Features**: `interactive`, `visualization`, `performance`

### Accessibility

#### Interactive Content
- **Keyboard navigation** for all interactive elements
- **Screen reader descriptions** for visualizations
- **Alternative text** for complex diagrams
- **Color-blind friendly** color schemes

```html
<!-- Good accessibility -->
<button aria-label="Start binary search animation" id="start-search">
    Start Search
</button>

<div role="img" aria-describedby="search-description">
    <!-- Visualization here -->
</div>
<div id="search-description" class="sr-only">
    Interactive binary search visualization showing array elements and search progress
</div>
```

### Performance Considerations

#### Large Datasets
```javascript
// Use requestAnimationFrame for smooth animations
function animate() {
    // Update visualization
    updateVisualization();
    
    if (animationRunning) {
        requestAnimationFrame(animate);
    }
}

// Throttle expensive operations
const throttledUpdate = _.throttle(updateVisualization, 16); // 60fps max
```

#### Memory Management
```javascript
// Clean up event listeners and animations
function cleanup() {
    // Remove event listeners
    document.removeEventListener('click', handleClick);
    
    // Cancel animations
    if (animationId) {
        cancelAnimationFrame(animationId);
    }
    
    // Clear intervals
    if (intervalId) {
        clearInterval(intervalId);
    }
}

// Call cleanup when leaving page
window.addEventListener('beforeunload', cleanup);
```

---

## Advanced Features

### Custom CSS for Specific Posts

You can include custom CSS directly in your markdown:

```html
<style>
.custom-algorithm-demo {
    /* Your custom styles here */
    background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
    padding: 2rem;
    border-radius: 12px;
    color: white;
}

.custom-algorithm-demo h3 {
    margin-top: 0;
}
</style>
```

### External Libraries

Include additional libraries as needed:

```html
<!-- For advanced math -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.11.0/math.min.js"></script>

<!-- For charts -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- For 3D visualizations -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

### Performance Monitoring

```javascript
// Monitor performance of interactive elements
const startTime = performance.now();

// Your algorithm/animation code

const endTime = performance.now();
console.log(`Animation took ${endTime - startTime} milliseconds`);
```

---

## Quick Reference

### Template Checklist

**Regular Post:**
- [ ] Correct filename format (`YYYY-MM-DD-title.md`)
- [ ] `layout: post` in frontmatter
- [ ] All required frontmatter fields
- [ ] Clear introduction and conclusion
- [ ] Code examples with syntax highlighting
- [ ] Mathematical analysis where relevant
- [ ] Real-world applications section

**Chapter Post:**
- [ ] `layout: chapter-post` in frontmatter
- [ ] Consistent `base_url` across all chapters
- [ ] Correct `current_chapter` and `total_chapters`
- [ ] Updated `chapter_titles` map
- [ ] Chapter-specific title and description

### Common Patterns

**Algorithm Complexity Table:**
```markdown
| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| Search    | O(log n)       | O(1)            |
| Insert    | O(log n)       | O(1)            |
| Delete    | O(log n)       | O(1)            |
```

**Before/After Code Comparison:**
```markdown
### ❌ Naive Approach
```python
# O(n²) solution
def find_duplicates(arr):
    duplicates = []
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            if arr[i] == arr[j]:
                duplicates.append(arr[i])
    return duplicates
```

### ✅ Optimized Approach
```python
# O(n) solution using hash set
def find_duplicates(arr):
    seen = set()
    duplicates = set()
    
    for num in arr:
        if num in seen:
            duplicates.add(num)
        seen.add(num)
    
    return list(duplicates)
```

---

## Conclusion

This guide provides everything you need to create engaging, educational content for Beyond Big-O. Remember:

1. **Always connect theory to practice**
2. **Include interactive elements** when possible
3. **Provide rigorous mathematical analysis**
4. **Show real-world applications**
5. **Write clean, production-ready code**
6. **Make content accessible** to all readers

The goal is to bridge the gap between competitive programming and software engineering, showing how algorithmic thinking scales from contest problems to production systems.

**Happy writing!** 🚀

---

*This guide will be updated as we add new features and learn from reader feedback. Have suggestions? Reach out at [anirudhsingh111100@gmail.com](mailto:anirudhsingh111100@gmail.com).*
