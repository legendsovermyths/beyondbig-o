---
layout: post
title: "Pattern Matching On Steroids: Searching Patterns In Billions Of Characters(With Fuzzy Matching)"
date: 2025-08-20
author: "Anirudh Singh"
excerpt: "Dive into the world of high-performance pattern matching where we tackle two real-world bioinformatics challenges: exact sequence matching across a billion characters, and fuzzy pattern matching with 2-3% tolerance."
tags: ["algorithms", "automata-theory", "fft", "text-processing", "aho-corasick", "interactive"]
reading_time: 30
---
## Introduction
Picture this: you're a computer scientist working at a fancy biotech company, sipping your third coffee of the morning, when you're presented with a problem that makes you question your life choices.

"We have two little problems for you," they say.

*Little problems.* Alright.

**Problem 1:** We have around 10,000 small DNA sequences, each about 100 bases long, and we need to find them in a sequence that's 10^9 characters long. Oh, and since these are small sequences, we want exact matches—no room for errors.

**Problem 2:** Our new Oxford Nanopore tech just spit out a read of 100,000 bases, and we need to match it against that same 10^9 sequence. But since this is a big read, we can tolerate 2-5% mismatches.

<div class="reality-check-box">
<strong>Reality Check:</strong> While these numbers are made up for this blog, they're actually very realistic in biotech! In genomics, you often have millions of short reads (100–300 bases) from sequencing, and you need to align them to a large genome (human ~3×10⁹ bases). Oxford Nanopore indeed produces long noisy reads (~10⁴–10⁵ bases). Since these reads are noisy, we need to account for a 2–3% error rate, which is why fuzzy matching with tolerance for mismatches is essential.
</div>

<style>
.reality-check-box {
    background-color: #f5f5f5 !important;
    padding: 1rem;
    border-radius: 8px;
    border-left: 4px solid #666 !important;
    margin: 1rem 0;
    color: inherit !important;
}

/* Explicit light mode */
@media (prefers-color-scheme: light) {
    .reality-check-box {
        background-color: #f5f5f5 !important;
        border-left-color: #666 !important;
        color: inherit !important;
    }
}

/* Dark mode only */
@media (prefers-color-scheme: dark) {
    .reality-check-box {
        background-color: #374151 !important;
        border-left-color: #9ca3af !important;
        color: #f3f4f6 !important;
    }
}

/* Fallback for explicit dark theme classes */
[data-theme="dark"] .reality-check-box,
.dark .reality-check-box {
    background-color: #374151 !important;
    border-left-color: #9ca3af !important;
    color: #f3f4f6 !important;
}

/* Explicit light theme classes */
[data-theme="light"] .reality-check-box,
.light .reality-check-box {
    background-color: #f5f5f5 !important;
    border-left-color: #666 !important;
    color: inherit !important;
}
</style>

Alright, let me translate this into something my CS brain can handle. We have a string of a billion characters, and we need to do two things:

1. **Search for 10,000 patterns** of length 100 in a 10^9 character long string
2. **Search for a single pattern** of length 100,000 in a 10^9 character long string with 2-5% mismatches allowed

Let me make this concrete with examples:

**Problem 1:** Imagine a 1 billion character long string like:
```
AGTCCGATCGTAGCTACGTAGCTACGTACGTACGTACGTACG...
```

And a dictionary of 10,000 patterns, each around 100 characters long:
```
AGTCCGATCGTAGCTACGTAGCTACGTACGTACGTACGTACG...  (pattern 1)
GTCCGATCGTAGCTACGTAGCTACGTACGTACGTACGTACGA...  (pattern 2)
TCCGATCGTAGCTACGTAGCTACGTACGTACGTACGTACGAT...  (pattern 3)
...
```

For each pattern in our dictionary, we need to find **exactly** where in this 1 billion string these patterns appear. No mismatches allowed.

**Problem 2:** Instead of a dictionary, we have one huge pattern of 100,000 characters:
```
AGTCCGATCGTAGCTACGTAGCTACGTACGTACGTACGTACG...  (continues for 100k chars)
```

We need to find where in our 1 billion string this pattern is present, but here's the thing: we also need to check where "almost" this pattern is present. So if in some places there's a `G` instead of `A` or a `T` instead of `C`, but the rest of the pattern aligns (around 97% of characters match), we'll consider it a match.

Simple, right? Just like finding a needle in a haystack, except the haystack is the size of a small country and you're looking for thousands of needles.

Let's start with the classic approach that every computer scientist learns in their first algorithms class: brute force! The "align and check" method that's so simple, it's almost beautiful.

For one pattern, we just align it at every position and see if it matches. This gives us `pattern_length × text_length` operations.

Let's do the math for the first problem:
- 10,000 patterns × 100 characters × 10^9 text length = **10^15 operations**

A modern computer can do roughly 10^8 operations per second, so:
- 10^15 ÷ 10^8 = 10^7 seconds
- 10^7 seconds = **116 days**

*116 days.* That's longer than most people's summer vacations. That's longer than some relationships. That's definitely longer than anyone will wait for results.

Now let's see for the other problem, this one's a bit more manageable (relatively speaking):
- 10^9 text length × 10^5 pattern length = **10^14 operations**
- 10^14 ÷ 10^8 = 10^6 seconds = **11.6 days**

Eleven and a half days. Still bad, but at least it's not measured in months. People might actually wait for this one. But it's still bad.

Okay, so brute force is out. But wait—Is there something that learnt while doing my Data Structures and Algorithms! Surely there's something useful there(otherwise what's the point)?

I know the KMP algorithm! That piece of algorithmic elegance that does pattern matching in O(n+m) time. That's linear time! That's amazing!

...except it only handles one pattern at a time, and it doesn't do fuzzy matching. It's like having a Ferrari that only works on perfectly paved roads and can only carry one passenger.

So here I am, with a problem that's way bigger than what my prep for this job covered. We're talking about processing billions of characters against thousands of patterns, with some needing fuzzy matching. This is definitely not your typical "find a word in a paragraph" homework problem.

What do I do now?

*Let me just run the brute force while I think of better solutions...*

## Enter the Trie
 
So I'm sitting there, watching my CPU slowly die, when I remembered something from my competitive programming days. There's this data structure called a trie — basically a tree where each path from root to leaf spells out a word.

But let me explain this properly because it's actually brilliant.

Imagine you have words like "CAT", "CAR", and "CARD". Instead of storing them separately, a trie builds a tree where they share common prefixes:

```
       ROOT
        |
        C
        |
        A
       / \
      R   T
      |   |
      ∅   ∅  (∅ means "end of word")
      |
      D
      |
      ∅
```

The magic happens when we search. Instead of checking "CAT", then "CAR", then "CARD" separately against our text, we walk through the text once and follow paths in the trie. When we see 'C', we go down the 'C' branch. Then 'A', down the 'A' branch. Then 'T'? We hit an end marker — boom, found "CAT"! But what if it was 'R' instead? We keep going and might find "CAR" or "CARD".

And then it hit me: "Wait, what if instead of checking each DNA sequence separately, I build one massive tree containing all 10,000 sequences and just walk through the billion-character string once?"

This is where things start getting interesting. Here's an interactive demo, but let's make it relevant to our problem. Instead of random words, let's use DNA-like sequences:
 
<div class="trie-demo">
    <div class="trie-controls">
        <div class="input-section">
            <input type="text" id="trie-word-input" placeholder="Type a DNA sequence (A, T, G, C)" maxlength="20">
            <button id="add-word-btn">Add Sequence</button>
            <button id="clear-trie-btn">Clear All</button>
        </div>
        <div class="search-section">
            <input type="text" id="search-input" placeholder="Search for a sequence" maxlength="20">
            <button id="search-btn">Search</button>
            <span id="search-result"></span>
        </div>
        <div class="preset-section">
            <button class="preset-btn" data-words="ATCG,ATCGA,ATCGAT">Add: ATCG, ATCGA, ATCGAT</button>
            <button class="preset-btn" data-words="GGCC,GGCCT,GGCCTA">Add: GGCC, GGCCT, GGCCTA</button>
            <button class="preset-btn" data-words="TACG,TACGA,TACGAC">Add: TACG, TACGA, TACGAC</button>
        </div>
    </div>
    <div id="trie-visualization"></div>
    <div id="trie-stats"></div>
</div>
 
<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js"></script>
<script>
class TrieNode {
    constructor(char = '') {
        this.char = char;
        this.children = new Map();
        this.isEndOfWord = false;
        this.id = Math.random().toString(36).substr(2, 9);
        this.x = 0;
        this.y = 0;
        this.targetX = 0;
        this.targetY = 0;
        this.highlighted = false;
        this.searchHighlight = false;
    }
}
 
class Trie {
    constructor() {
        this.root = new TrieNode();
        this.wordCount = 0;
        this.searchAnimation = {
            active: false,
            path: [],
            currentIndex: 0,
            speed: 30 // frames per step
        };
    }
 
    insert(word) {
        let node = this.root;
        word = word.toLowerCase().trim();
        
        if (!word) return false;
        
        for (let char of word) {
            if (!node.children.has(char)) {
                node.children.set(char, new TrieNode(char));
            }
            node = node.children.get(char);
        }
        
        if (!node.isEndOfWord) {
            node.isEndOfWord = true;
            this.wordCount++;
            return true;
        }
        return false;
    }
 
    search(word) {
        let node = this.root;
        word = word.toLowerCase().trim();
        
        for (let char of word) {
            if (!node.children.has(char)) {
                return false;
            }
            node = node.children.get(char);
        }
        
        return node.isEndOfWord;
    }
 
    getAllNodes() {
        const nodes = [];
        const edges = [];
        
        const traverse = (node, parent = null) => {
            nodes.push(node);
            if (parent) {
                edges.push({ from: parent.id, to: node.id });
            }
            
            for (let child of node.children.values()) {
                traverse(child, node);
            }
        };
        
        traverse(this.root);
        return { nodes, edges };
    }
 
    highlightPath(word) {
        this.clearHighlights();
        return this.animateSearch(word);
    }
 
    animateSearch(word) {
        word = word.toLowerCase().trim();
        const path = [this.root];
        let node = this.root;
        let valid = true;
        
        for (let char of word) {
            if (!node.children.has(char)) {
                valid = false;
                break;
            }
            node = node.children.get(char);
            path.push(node);
        }
        
        this.searchAnimation = {
            active: true,
            path: path,
            currentIndex: 0,
            speed: 20,
            frameCounter: 0,
            valid: valid,
            isComplete: false,
            foundWord: valid && node.isEndOfWord
        };
        
        return valid && node.isEndOfWord;
    }
 
    updateSearchAnimation() {
        if (!this.searchAnimation.active) return;
        
        this.searchAnimation.frameCounter++;
        
        if (this.searchAnimation.frameCounter >= this.searchAnimation.speed) {
            this.searchAnimation.frameCounter = 0;
            
            if (this.searchAnimation.currentIndex < this.searchAnimation.path.length) {
                // Highlight current node
                this.searchAnimation.path[this.searchAnimation.currentIndex].searchHighlight = true;
                this.searchAnimation.currentIndex++;
            } else {
                // Animation complete
                this.searchAnimation.active = false;
                this.searchAnimation.isComplete = true;
            }
        }
    }
 
    clearHighlights() {
        const { nodes } = this.getAllNodes();
        nodes.forEach(node => {
            node.highlighted = false;
            node.searchHighlight = false;
        });
        this.searchAnimation = {
            active: false,
            path: [],
            currentIndex: 0,
            speed: 20,
            frameCounter: 0,
            valid: false,
            isComplete: false,
            foundWord: false
        };
    }
}
 
let trieSketch = function(p) {
    let trie = new Trie();
    let animationSpeed = 0.1;
    let colors = {
        bg: '#ffffff',
        accent: '#111111',
        text: '#111111'
    };
    
    p.setup = function() {
        // Create initial canvas
        let canvas = p.createCanvas(800, 500);
        canvas.parent('trie-visualization');
        
        // Enable smooth rendering and high DPI
        p.pixelDensity(Math.min(window.devicePixelRatio || 1, 2));
        p.smooth();
        
        // Force high-quality rendering
        canvas.canvas.style.imageRendering = 'auto';
        canvas.canvas.style.imageRendering = 'high-quality';
        
        // Add some initial DNA sequences
        console.log('Adding initial DNA sequences to trie...');
        trie.insert('ATCG');
        trie.insert('ATCGA');
        trie.insert('ATCGAT');
        console.log('Trie sequence count:', trie.wordCount);
        updateTrieStats();
        
        // Update colors after DOM is ready
        setTimeout(updateColors, 100);
    };
    
    function updateColors() {
        try {
            // Get the actual computed CSS colors
            const computedStyle = getComputedStyle(document.documentElement);
            const bgPrimary = computedStyle.getPropertyValue('--bg-primary').trim();
            const textPrimary = computedStyle.getPropertyValue('--text-primary').trim();
            
            // Use the theme colors or fallback colors
            if (bgPrimary && textPrimary) {
                colors.bg = bgPrimary;
                colors.text = textPrimary;
                colors.accent = textPrimary;
            } else {
                // Better fallback detection - check if body background is dark
                const bodyBg = getComputedStyle(document.body).backgroundColor;
                const isDark = bodyBg && (bodyBg.includes('rgb(') ? 
                    bodyBg.match(/\d+/g).reduce((sum, val) => sum + parseInt(val), 0) < 384 : 
                    false);
                
                if (isDark) {
                    colors.bg = '#1f2937';
                    colors.text = '#f9fafb';
                    colors.accent = '#f9fafb';
                } else {
                    colors.bg = '#ffffff';
                    colors.text = '#1f2937';
                    colors.accent = '#1f2937';
                }
            }
            
            console.log('Updated trie colors:', colors);
        } catch (e) {
            console.log('Using fallback colors for trie');
            colors.bg = '#ffffff';
            colors.text = '#1f2937';
            colors.accent = '#1f2937';
        }
    }
    
    p.draw = function() {
        // Update colors on each frame to respond to theme changes
        updateColors();
        
        p.background(colors.bg);
        
        // Enable smooth rendering for this frame
        p.smooth();
        
        // Update search animation
        trie.updateSearchAnimation();
        
        // Calculate positions
        calculatePositions();
        
        // Draw the trie
        drawTrie();
        
        // Update positions with smooth animation
        animateNodes();
        
        // Update search result if animation just completed
        if (trie.searchAnimation.isComplete) {
            updateSearchResultFromAnimation();
            trie.searchAnimation.isComplete = false;
        }
    };
    
    function calculatePositions() {
        const { nodes } = trie.getAllNodes();
        const levels = new Map();
        
        // Group nodes by level
        const assignLevels = (node, level = 0) => {
            if (!levels.has(level)) levels.set(level, []);
            levels.get(level).push(node);
            
            for (let child of node.children.values()) {
                assignLevels(child, level + 1);
            }
        };
        
        assignLevels(trie.root);
        
        // Calculate dynamic scaling
        const maxLevel = Math.max(...levels.keys());
        const maxNodesInLevel = Math.max(...Array.from(levels.values()).map(arr => arr.length));
        
        // Auto-resize canvas if needed
        resizeCanvasIfNeeded(maxLevel, maxNodesInLevel);
        
        // Balanced spacing - not too cramped, not too stretched
        const availableHeight = p.height - 160; // Space for legend
        const availableWidth = p.width - 100; // Horizontal padding
        const levelHeight = Math.max(70, Math.min(90, availableHeight / (maxLevel + 1))); // Moderate level height
        const minNodeSpacing = Math.max(80, availableWidth / Math.max(maxNodesInLevel, 4)); // Reasonable spacing
        
        for (let [level, levelNodes] of levels.entries()) {
            const y = 60 + level * levelHeight; // Not too high, not too low
            
            if (levelNodes.length === 1) {
                levelNodes[0].targetX = p.width / 2;
                levelNodes[0].targetY = y;
            } else {
                // Calculate comfortable spacing without being excessive
                const nodeSpacing = Math.max(minNodeSpacing, 90); // Moderate minimum spacing
                const totalWidth = (levelNodes.length - 1) * nodeSpacing;
                const maxAllowedWidth = availableWidth;
                
                // Scale down if needed but maintain reasonable minimum
                const actualSpacing = totalWidth > maxAllowedWidth ? 
                    Math.max(70, maxAllowedWidth / (levelNodes.length - 1)) : nodeSpacing;
                
                const actualTotalWidth = (levelNodes.length - 1) * actualSpacing;
                const startX = p.width / 2 - actualTotalWidth / 2;
                
                levelNodes.forEach((node, index) => {
                    node.targetX = startX + index * actualSpacing;
                    node.targetY = y;
                });
            }
        }
    }
    
    function resizeCanvasIfNeeded(maxLevel, maxNodesInLevel) {
        // More conservative sizing - only resize when really needed
        const minWidth = Math.max(600, maxNodesInLevel * 90 + 160); // Reduced from 120px to 90px per node
        const minHeight = Math.max(400, maxLevel * 70 + 160); // Reduced from 90px to 70px per level
        
        // Get container constraints
        const container = document.getElementById('trie-visualization');
        const containerWidth = container.clientWidth;
        const maxWidth = Math.min(900, containerWidth - 40); // Reduced max width from 1200 to 900
        const maxHeight = 600; // Reduced max height from 800 to 600
        
        // Calculate optimal dimensions - more conservative growth
        const targetWidth = Math.min(maxWidth, Math.max(800, minWidth));
        const targetHeight = Math.min(maxHeight, Math.max(500, minHeight));
        
        // Only resize if significantly different (avoid constant small adjustments)
        const widthDiff = Math.abs(p.width - targetWidth);
        const heightDiff = Math.abs(p.height - targetHeight);
        
        if (widthDiff > 80 || heightDiff > 80) { // Increased threshold from 50 to 80
            console.log(`Resizing canvas from ${p.width}x${p.height} to ${targetWidth}x${targetHeight}`);
            
            // Resize canvas
            p.resizeCanvas(targetWidth, targetHeight);
            
            // Update canvas styling for responsiveness
            const canvas = p.canvas;
            canvas.style.maxWidth = '100%';
            canvas.style.height = 'auto';
            
            // If content is too wide, enable horizontal scrolling on the container
            if (targetWidth > containerWidth - 40) {
                container.style.overflowX = 'auto';
                container.style.overflowY = 'hidden';
            } else {
                container.style.overflowX = 'visible';
                container.style.overflowY = 'visible';
            }
        }
    }
    
    function animateNodes() {
        const { nodes } = trie.getAllNodes();
        
        nodes.forEach(node => {
            node.x += (node.targetX - node.x) * animationSpeed;
            node.y += (node.targetY - node.y) * animationSpeed;
        });
    }
    
    function drawTrie() {
        const { nodes, edges } = trie.getAllNodes();
        
        // Debug info
        if (nodes.length === 0) {
            p.fill('#ef4444');
            p.textAlign(p.CENTER, p.CENTER);
            p.textSize(16);
            p.text('No nodes found - check console for errors', p.width/2, p.height/2);
            return;
        }
        
        // Draw edges first
        edges.forEach(edge => {
            const fromNode = nodes.find(n => n.id === edge.from);
            const toNode = nodes.find(n => n.id === edge.to);
            
            if (fromNode && toNode) {
                // Subtle green for animated search, otherwise use theme colors
                p.stroke(toNode.searchHighlight ? '#16a34a' : colors.text);
                p.strokeWeight(toNode.searchHighlight ? 2.5 : 1.5);
                p.line(fromNode.x, fromNode.y, toNode.x, toNode.y);
            }
        });
        
        // Draw nodes
        nodes.forEach(node => {
            // Node circle
            if (node.searchHighlight) {
                p.fill('#d1fae5'); // very subtle green fill
                p.stroke('#16a34a'); // subtle green outline
            } else if (node.isEndOfWord) {
                p.fill(colors.bg);
                p.stroke(colors.text);
            } else if (node === trie.root) {
                p.fill(colors.bg);
                p.stroke(colors.text);
            } else {
                p.fill(colors.bg);
                p.stroke(colors.text);
            }
            
            p.strokeWeight(1.5);
            p.circle(node.x, node.y, node === trie.root ? 40 : 35);
            
            // Node label
            p.fill(colors.text);
            p.textAlign(p.CENTER, p.CENTER);
            p.textSize(node === trie.root ? 16 : 14);
            p.textStyle(p.BOLD);
            p.text(node === trie.root ? 'ROOT' : node.char.toUpperCase(), node.x, node.y);
            
            // End of word indicator
            if (node.isEndOfWord && node !== trie.root) {
                p.fill(colors.text);
                p.noStroke();
                p.circle(node.x + 12, node.y - 12, 8);
            }
        });
        
        // Draw legend
        drawLegend();
    }
    
    function drawLegend() {
        const legendX = 20;
        const legendY = p.height - 120;
        
        // Determine if we're in light or dark theme more reliably
        const isLightTheme = isLightMode(colors.bg, colors.text);
        
        // Legend background - use proper contrast for each theme
        const legendBg = isLightTheme ? '#ffffff' : '#374151';
        const legendBorder = isLightTheme ? '#d1d5db' : '#6b7280';
        const legendTextColor = isLightTheme ? '#1f2937' : '#f9fafb';
        
        p.fill(legendBg);
        p.stroke(legendBorder);
        p.strokeWeight(1);
        p.rect(legendX - 10, legendY - 10, 190, 100, 8);
        
        p.fill(legendTextColor);
        p.textAlign(p.LEFT);
        p.textSize(12);
        p.noStroke();
        
        // Node fill colors - use theme-appropriate colors
        const nodeFillColor = isLightTheme ? '#f9fafb' : '#4b5563';
        const nodeStrokeColor = legendTextColor;
        
        // Root node
        p.fill(nodeFillColor);
        p.stroke(nodeStrokeColor);
        p.strokeWeight(1);
        p.circle(legendX + 10, legendY + 5, 15);
        p.noStroke();
        p.fill(legendTextColor);
        p.text('Root node', legendX + 30, legendY + 8);
        
        // Regular node
        p.fill(nodeFillColor);
        p.stroke(nodeStrokeColor);
        p.strokeWeight(1);
        p.circle(legendX + 10, legendY + 25, 15);
        p.noStroke();
        p.fill(legendTextColor);
        p.text('Character node', legendX + 30, legendY + 28);
        
        // End of word node
        p.fill(nodeFillColor);
        p.stroke(nodeStrokeColor);
        p.strokeWeight(1);
        p.circle(legendX + 10, legendY + 45, 15);
        // Small dot to indicate end of word
        p.noStroke();
        p.fill(legendTextColor);
        p.circle(legendX + 10, legendY + 45, 4);
        p.text('End of word', legendX + 30, legendY + 48);
        
        // Search highlight - use theme-aware colors
        const searchFillColor = isLightTheme ? '#d1fae5' : '#065f46';
        const searchStrokeColor = isLightTheme ? '#16a34a' : '#10b981';
        
        p.fill(searchFillColor);
        p.stroke(searchStrokeColor);
        p.strokeWeight(1);
        p.circle(legendX + 10, legendY + 65, 15);
        p.noStroke();
        p.fill(legendTextColor);
        p.text('Search path', legendX + 30, legendY + 68);
    }
    
    // Helper function to detect light mode more reliably
    function isLightMode(bgColor, textColor) {
        // Check if we have valid colors
        if (!bgColor || !textColor) return true; // Default to light mode
        
        // If colors are hex codes, convert to check brightness
        const getBrightness = (color) => {
            if (color.startsWith('#')) {
                const hex = color.slice(1);
                const r = parseInt(hex.substr(0, 2), 16);
                const g = parseInt(hex.substr(2, 2), 16);
                const b = parseInt(hex.substr(4, 2), 16);
                return (r * 299 + g * 587 + b * 114) / 1000;
            }
            // For named colors or other formats, use simple heuristics
            return bgColor.includes('white') || bgColor.includes('#fff') || 
                   bgColor.includes('#f') || textColor.includes('black') || 
                   textColor.includes('#000') || textColor.includes('#1') ? 255 : 50;
        };
        
        const bgBrightness = getBrightness(bgColor);
        return bgBrightness > 128; // Light if background brightness > 128
    }
    
    // Global functions
    window.addWordToTrie = function(word) {
        if (trie.insert(word)) {
            updateTrieStats();
            // Force recalculation of positions and potential resize
            setTimeout(() => {
                calculatePositions();
            }, 100);
            return true;
        }
        return false;
    };
    
    window.searchInTrie = function(word) {
        const found = trie.highlightPath(word);
        updateSearchResult(word, found);
        return found;
    };
    
    window.clearTrie = function() {
        trie = new Trie();
        updateTrieStats();
        clearSearchResult();
        // Reset canvas to default size when cleared
        setTimeout(() => {
            if (p.width !== 800 || p.height !== 500) {
                console.log('Resetting canvas to default size');
                p.resizeCanvas(800, 500);
                const container = document.getElementById('trie-visualization');
                container.style.overflowX = 'visible';
                container.style.overflowY = 'visible';
            }
        }, 100);
    };
    
    function updateTrieStats() {
        const { nodes } = trie.getAllNodes();
        document.getElementById('trie-stats').innerHTML = `
            <strong>DNA sequences stored:</strong> ${trie.wordCount} | 
            <strong>Nodes created:</strong> ${nodes.length - 1} | 
            <strong>Memory saved:</strong> ${calculateMemorySaving()}%
        `;
    }
    
    function calculateMemorySaving() {
        if (trie.wordCount === 0) return 0;
        const { nodes } = trie.getAllNodes();
        const trieNodes = nodes.length - 1; // Exclude root
        const naiveStorage = trie.wordCount * 8; // Assume avg 8 chars per DNA sequence
        const saving = Math.max(0, Math.round((1 - trieNodes / naiveStorage) * 100));
        return saving;
    }
    
    function updateSearchResult(word, found) {
        const resultEl = document.getElementById('search-result');
        if (found) {
            resultEl.innerHTML = `"${word}" found!`;
            resultEl.style.color = '#22c55e';
        } else {
            resultEl.innerHTML = `"${word}" not found`;
            resultEl.style.color = '#ef4444';
        }
    }
 
    function updateSearchResultFromAnimation() {
        const resultEl = document.getElementById('search-result');
        const word = document.getElementById('search-input').value.trim();
        
        if (trie.searchAnimation.foundWord) {
            resultEl.innerHTML = `"${word}" found!`;
            resultEl.style.color = '#22c55e';
        } else if (trie.searchAnimation.valid) {
            resultEl.innerHTML = `"${word}" is a prefix but not a complete word`;
            resultEl.style.color = '#f59e0b';
        } else {
            resultEl.innerHTML = `"${word}" not found`;
            resultEl.style.color = '#ef4444';
        }
    }
    
    function clearSearchResult() {
        document.getElementById('search-result').innerHTML = '';
        trie.clearHighlights();
    }
};
 
// Initialize p5 sketch
try {
    new p5(trieSketch);
    console.log('Trie sketch initialized successfully');
} catch (error) {
    console.error('Error initializing trie sketch:', error);
}
 
// Event listeners
document.addEventListener('DOMContentLoaded', function() {
    console.log('DOM loaded, setting up event listeners...');
    
    const addBtn = document.getElementById('add-word-btn');
    if (!addBtn) {
        console.error('Add button not found!');
        return;
    }
    
    addBtn.addEventListener('click', function() {
        const input = document.getElementById('trie-word-input');
        const word = input.value.trim();
        
        if (word) {
            if (window.addWordToTrie(word)) {
                input.value = '';
                input.focus();
            } else {
                alert('Sequence already exists in trie!');
            }
        }
    });
 
    document.getElementById('search-btn').addEventListener('click', function() {
        const input = document.getElementById('search-input');
        const word = input.value.trim();
        
        if (word) {
            window.searchInTrie(word);
        }
    });
 
    document.getElementById('clear-trie-btn').addEventListener('click', function() {
        window.clearTrie();
        document.getElementById('search-input').value = '';
        document.getElementById('trie-word-input').value = '';
    });
 
    // Preset buttons
    document.querySelectorAll('.preset-btn').forEach(btn => {
        btn.addEventListener('click', function() {
            const words = this.dataset.words.split(',');
            words.forEach(word => window.addWordToTrie(word.trim()));
        });
    });
 
    // Enter key support
    document.getElementById('trie-word-input').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') {
            document.getElementById('add-word-btn').click();
        }
    });
 
    document.getElementById('search-input').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') {
            document.getElementById('search-btn').click();
        }
    });
 
    // Clear search highlights when typing
    document.getElementById('search-input').addEventListener('input', function() {
        if (window.clearTrie) {
            document.getElementById('search-result').innerHTML = '';
            // Don't clear highlights while typing for better UX
        }
    });
});
</script>
 
<style>
.trie-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-color);
}

.trie-controls {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-bottom: 2rem;
}

.input-section, .search-section, .preset-section {
    display: flex;
    gap: 1rem;
    align-items: center;
    flex-wrap: wrap;
}

.trie-controls input[type="text"] {
    padding: 0.5rem 1rem;
    border: 2px solid var(--border-primary);
    border-radius: 6px;
    background: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    min-width: 200px;
    flex: 1;
    min-width: 150px;
}

.trie-controls input[type="text"]:focus {
    outline: none;
    border-color: var(--accent-primary);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.trie-controls button {
    padding: 0.5rem 1rem;
    background: transparent;
    color: var(--text-primary);
    border: 1.5px solid var(--text-primary);
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    font-size: 14px;
    transition: all 0.2s ease;
    white-space: nowrap;
}

.trie-controls button:hover {
    background: var(--text-primary);
    color: var(--bg-primary);
    transform: translateY(-1px);
}

.trie-controls button:active {
    transform: translateY(0);
}

.preset-btn {
    font-size: 12px !important;
    padding: 0.4rem 0.8rem !important;
}

#clear-trie-btn {
    border-color: #dc2626;
    color: #dc2626;
}

#clear-trie-btn:hover {
    background: #dc2626;
    color: #ffffff;
}

#search-result {
    font-weight: 600;
    font-size: 14px;
    color: var(--text-primary);
}

#trie-visualization {
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
    margin-bottom: 1rem;
    padding: 1rem;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 500px;
    overflow: auto; /* Allow both horizontal and vertical scrolling when needed */
    position: relative;
}

#trie-visualization canvas {
    display: block;
    image-rendering: auto !important;
    image-rendering: -webkit-optimize-quality !important;
    image-rendering: smooth !important;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    border-radius: 4px;
    max-width: 100%;
    height: auto;
}

#trie-stats {
    text-align: center;
    font-size: 14px;
    color: var(--text-secondary);
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
}

/* Mobile responsiveness for trie demo */
@media (max-width: 768px) {
    .trie-demo {
        padding: 1rem;
        margin: 1.5rem 0;
    }
    
    .input-section, .search-section, .preset-section {
        flex-direction: column;
        align-items: stretch;
        gap: 0.75rem;
    }
    
    .trie-controls input[type="text"] {
        min-width: 100%;
        font-size: 16px; /* Prevents zoom on iOS */
    }
    
    .trie-controls button {
        width: 100%;
        justify-content: center;
    }
    
    .preset-btn {
        font-size: 11px !important;
        padding: 0.5rem !important;
    }
    
    #trie-visualization {
        min-height: 400px;
        padding: 0.5rem;
        overflow-x: auto; /* Always allow horizontal scroll on mobile */
    }
    
    #trie-stats {
        font-size: 12px;
        padding: 0.75rem;
    }
}

@media (max-width: 480px) {
    .trie-demo {
        padding: 0.75rem;
    }
    
    #trie-visualization {
        min-height: 350px;
    }
    
    .preset-btn {
        font-size: 10px !important;
    }
}
</style>
 
Pretty neat, right? You can see how sequences like "ATCG", "ATCGA", and "ATCGAT" share the common prefix "ATCG" and the trie automatically reuses those nodes. That's memory efficiency in action!

Now imagine scaling this up to our actual problem: 10,000 DNA sequences stored in one massive trie. Instead of checking each sequence individually against our billion-character string, we walk through the string once, character by character, following paths in the trie.

Simple, right?
 
Well... no. Not really.
 
## Wait, There's a Problem

Now here's the thing about tries: they're absolutely amazing for checking if a word exists in a dictionary. Got a word? Walk down the trie, hit an end marker, boom - it's there. Beautiful.

But our problem is different. We're not asking "does this word exist?" We're asking "does our billion-character DNA sequence contain any of these 10,000 patterns?" And that's where tries get... problematic.

Let me walk you through what actually happens when you use a basic trie for text searching. Say we've got DNA sequences "ATCGA" and "TCGAT" in our dictionary, and we're searching through the text "ATCGC".

Here's how it plays out:
1. Start at root
2. See 'A' → follow 'A' path
3. See 'T' → follow 'T' path  
4. See 'C' → follow 'C' path
5. See 'G' → follow 'G' path
6. See 'C' → **no 'C' transition available from this node**
7. Back to root we go

But hold on. We just processed "ATCG" and then gave up when we hit that final 'C'. What about "TCGAT"? That could totally be a match starting from position 2 in our text! But our trie already moved past the 'T' at position 2.

So we're not actually done with those characters. We need to start over from position 2 and check "TCGC", then position 3 and check "CGC", then position 4... you get the idea.

This means we're not really getting that beautiful single-pass behavior we wanted. We're still doing multiple passes, just in a slightly smarter way.

And it gets worse with overlapping gene sequences. Imagine we're looking for both "GAT" and "CGAT" in the text "CGATCG". Our trie will happily find "CGAT" but completely miss that "GAT" was sitting right there in positions 2-4. And we will have to process that position again to catch that pattern.

That's when I realized: the trie is great for dictionary lookups, but for multi-pattern match we need something more powerful.
 
## Enter Aho-Corasick
 
This is where Aho and Corasick showed up in 1975 and basically said, "You know what? Let's fix this properly."

Picture this: you're walking through the trie, following the path for "ATCGA", and suddenly you hit a character that doesn't match. Instead of giving up like a quitter and trudging all the way back to the root, what if the trie could whisper: "Hey, I know you were trying to match 'ATCGA', but look at the last few characters you've seen... 'TCGA'... doesn't that look like the beginning of something else in our dictionary?"

That's the genius of Aho-Corasick. It builds these magical shortcuts called **failure links** that are basically the algorithm's way of saying "this path didn't work out, but here's where you should jump to continue your quest."

Here's how failure links work: Say you're looking for patterns "ATCG" and "TCGA" in the text "ATCGA". You start matching "ATCG" perfectly... A-T-C-G... but then you hit 'A' and there's no path for "ATCGA" in your trie.

Instead of starting over from the beginning, the failure link says: "Wait! Look at what you just read: 'TCGA'. That's exactly the pattern 'TCGA' you're also looking for!" So it instantly jumps you to the end of the "TCGA" pattern match.

Without failure links, you'd miss this overlap completely and would need to start over. With them, you catch every possible match in one pass.

The key insight is that failure links always jump to the **longest matching suffix**. When you've read "ATCG" and hit a dead end, the algorithm looks at all the suffixes of what you've seen: "TCG", "CG", "G". It finds the longest one that's also a prefix of some pattern in your dictionary. In our case, "TCG" is the start of "TCGA", so the failure link jumps you there, and you can immediately continue matching from that point.

So in a way failure links help you save all the progress that you made till this point.

Let me show you how this works with some DNA sequences that actually demonstrate the overlap problem:
 
<div class="aho-corasick-demo">
    <div class="ac-controls">
        <div class="input-section">
            <input type="text" id="ac-text-input" placeholder="Type DNA sequence to search through" maxlength="50" value="ATCGATCG">
            <button id="ac-search-btn">Search</button>
            <button id="ac-step-btn">Step Through</button>
            <button id="ac-reset-btn">Reset</button>
        </div>
        <div class="pattern-section">
            <button class="pattern-btn" data-patterns="TCG,ATCG">Patterns: TCG, ATCG (overlapping!)</button>
            <button class="pattern-btn" data-patterns="GAT,CGAT">Patterns: GAT, CGAT (classic overlap)</button>
            <button class="pattern-btn" data-patterns="ATC,TCGA,CGAT">Patterns: ATC, TCGA, CGAT (multiple overlaps)</button>
        </div>
        <div class="status-section">
            <span id="ac-status">Click a pattern set to begin</span>
            <span id="ac-matches"></span>
        </div>
    </div>
    <div id="aho-corasick-visualization"></div>
    <div class="text-display">
        <div id="text-visualization"></div>
    </div>
</div>
 
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
class ACNode {
    constructor(char = '') {
        this.char = char;
        this.children = new Map();
        this.isEndOfWord = false;
        this.failureLink = null;
        this.outputLinks = [];
        this.id = Math.random().toString(36).substr(2, 9);
        this.x = 0;
        this.y = 0;
        this.targetX = 0;
        this.targetY = 0;
        this.depth = 0;
        this.highlighted = false;
        this.currentMatch = false;
    }
}
 
class AhoCorasick {
    constructor() {
        this.root = new ACNode();
        this.patterns = [];
        this.currentState = this.root;
        this.textPosition = 0;
        this.matches = [];
        this.stepMode = false;
        this.animationState = {
            isAnimating: false,
            currentChar: '',
            previousState: null,
            usedFailureLink: false,
            newMatches: []
        };
    }
 
    addPattern(pattern) {
        this.patterns.push(pattern);
        let node = this.root;
        
        for (let char of pattern.toLowerCase()) {
            if (!node.children.has(char)) {
                node.children.set(char, new ACNode(char));
            }
            node = node.children.get(char);
        }
        
        node.isEndOfWord = true;
        node.pattern = pattern;
    }
 
    buildFailureLinks() {
        // BFS to build failure links
        const queue = [];
        
        // All first level nodes have failure link to root
        for (let child of this.root.children.values()) {
            child.failureLink = this.root;
            child.depth = 1;
            queue.push(child);
        }
        
        while (queue.length > 0) {
            const current = queue.shift();
            
            for (let [char, child] of current.children.entries()) {
                queue.push(child);
                child.depth = current.depth + 1;
                
                // Find failure link
                let failureState = current.failureLink;
                
                while (failureState !== null && !failureState.children.has(char)) {
                    failureState = failureState.failureLink;
                }
                
                if (failureState === null) {
                    child.failureLink = this.root;
                } else {
                    child.failureLink = failureState.children.get(char);
                }
                
                // Build output links
                child.outputLinks = [];
                let outputState = child.failureLink;
                while (outputState !== null) {
                    if (outputState.isEndOfWord) {
                        child.outputLinks.push(outputState);
                    }
                    outputState = outputState.failureLink;
                }
            }
        }
    }
 
    search(text, stepMode = false) {
        this.matches = [];
        this.currentState = this.root;
        this.textPosition = 0;
        this.stepMode = stepMode;
        
        if (!stepMode) {
            for (let i = 0; i < text.length; i++) {
                this.processCharacter(text[i].toLowerCase(), i);
            }
        }
        
        return this.matches;
    }
    
    processCharacter(char, position) {
        // Normalize character to lowercase
        char = char.toLowerCase();
        
        this.animationState.currentChar = char;
        this.animationState.previousState = this.currentState;
        this.animationState.usedFailureLink = false;
        this.animationState.newMatches = [];
        
        // Clear previous match highlights but keep path highlights for step mode
        const { nodes } = this.getAllNodes();
        nodes.forEach(node => {
            node.currentMatch = false;
        });
        
        // Try to find transition
        let originalState = this.currentState;
        while (this.currentState !== null && !this.currentState.children.has(char)) {
            this.currentState = this.currentState.failureLink;
            this.animationState.usedFailureLink = true;
        }
        
        if (this.currentState === null) {
            this.currentState = this.root;
        } else {
            this.currentState = this.currentState.children.get(char);
        }
        
        // Clear all highlights first, then highlight current path
        const allNodes = this.getAllNodes().nodes;
        allNodes.forEach(node => node.highlighted = false);
        
        // Highlight current path
        this.currentState.highlighted = true;
        
        // Check for matches
        if (this.currentState.isEndOfWord) {
            this.currentState.currentMatch = true;
            const match = {
                pattern: this.currentState.pattern,
                position: position - this.currentState.pattern.length + 1,
                endPosition: position
            };
            this.matches.push(match);
            this.animationState.newMatches.push(match);
        }
        
        // Check output links for additional matches
        for (let outputNode of this.currentState.outputLinks) {
            outputNode.currentMatch = true;
            const match = {
                pattern: outputNode.pattern,
                position: position - outputNode.pattern.length + 1,
                endPosition: position
            };
            this.matches.push(match);
            this.animationState.newMatches.push(match);
        }
    }
 
    getAllNodes() {
        const nodes = [];
        const edges = [];
        const failureLinks = [];
        
        const traverse = (node, parent = null) => {
            nodes.push(node);
            
            if (parent) {
                edges.push({ 
                    from: parent.id, 
                    to: node.id, 
                    type: 'trie',
                    char: node.char 
                });
            }
            
            if (node.failureLink && node.failureLink !== this.root) {
                failureLinks.push({
                    from: node.id,
                    to: node.failureLink.id,
                    type: 'failure'
                });
            }
            
            for (let child of node.children.values()) {
                traverse(child, node);
            }
        };
        
        traverse(this.root);
        return { nodes, edges, failureLinks };
    }
 
    clearHighlights() {
        const { nodes } = this.getAllNodes();
        nodes.forEach(node => {
            node.highlighted = false;
            node.currentMatch = false;
        });
    }
}
 
// Visualization setup
let ahoCorasickViz = {
    ac: new AhoCorasick(),
    svg: null,
    width: 800,
    height: 400,
    stepMode: false,
    currentText: '',
    currentPosition: 0,
    
    init() {
        this.svg = d3.select("#aho-corasick-visualization")
            .append("svg")
            .attr("width", this.width)
            .attr("height", this.height)
            .style("background", "var(--bg-primary)")
            .style("border-radius", "8px")
            .style("border", "1px solid var(--border-primary)");
        
        this.updateStatus("Click a pattern set to begin");
    },
    
    setPatterns(patterns) {
        this.ac = new AhoCorasick();
        patterns.forEach(p => this.ac.addPattern(p));
        this.ac.buildFailureLinks();
        this.render();
        this.updateStatus(`Loaded patterns: ${patterns.join(', ')}`);
    },
    
    render() {
        const { nodes, edges, failureLinks } = this.ac.getAllNodes();
        
        // Calculate positions
        this.calculatePositions(nodes);
        
        // Clear previous render
        this.svg.selectAll("*").remove();
        
        // Update SVG background to ensure it reflects current theme
        this.svg.style("background", "var(--bg-primary)")
               .style("border", "1px solid var(--border-primary)");
        
        // Draw failure links first (dashed lines)
        this.svg.selectAll("line.failure")
            .data(failureLinks)
            .enter()
            .append("line")
            .attr("class", "failure")
            .attr("x1", d => nodes.find(n => n.id === d.from).x)
            .attr("y1", d => nodes.find(n => n.id === d.from).y)
            .attr("x2", d => nodes.find(n => n.id === d.to).x)
            .attr("y2", d => nodes.find(n => n.id === d.to).y)
            .attr("stroke", "#ef4444")
            .attr("stroke-width", 1.5)
            .attr("stroke-dasharray", "5,5")
            .attr("opacity", 0.7);
        
        // Draw trie edges
        this.svg.selectAll("line.trie")
            .data(edges)
            .enter()
            .append("line")
            .attr("class", "trie")
            .attr("x1", d => nodes.find(n => n.id === d.from).x)
            .attr("y1", d => nodes.find(n => n.id === d.from).y)
            .attr("x2", d => nodes.find(n => n.id === d.to).x)
            .attr("y2", d => nodes.find(n => n.id === d.to).y)
            .attr("stroke", "var(--text-primary)")
            .attr("stroke-width", 2);
        
        // Draw nodes
        this.svg.selectAll("circle")
            .data(nodes)
            .enter()
            .append("circle")
            .attr("cx", d => d.x)
            .attr("cy", d => d.y)
            .attr("r", d => d === this.ac.root ? 20 : 18)
            .attr("fill", d => {
                if (d.currentMatch) return "#22c55e";
                if (d.highlighted) return "#d1fae5";
                if (d.isEndOfWord) return "var(--bg-primary)";
                return "var(--bg-primary)";
            })
            .attr("stroke", d => {
                if (d.currentMatch) return "#16a34a";
                if (d.highlighted) return "#16a34a";
                return "var(--text-primary)";
            })
            .attr("stroke-width", 2);
        
        // Get current theme colors - force color refresh for theme compatibility
        const computedStyle = getComputedStyle(document.documentElement);
        const textColor = computedStyle.getPropertyValue('--text-primary').trim() || '#1f2937';
        const secondaryTextColor = computedStyle.getPropertyValue('--text-secondary').trim() || '#6b7280';
        
        // Add node labels
        this.svg.selectAll("text")
            .data(nodes)
            .enter()
            .append("text")
            .attr("x", d => d.x)
            .attr("y", d => d.y + 5)
            .attr("text-anchor", "middle")
            .attr("fill", textColor)
            .attr("font-weight", "bold")
            .attr("font-size", "12px")
            .text(d => d === this.ac.root ? 'ROOT' : d.char.toUpperCase());
        
        // Add end-of-word indicators
        this.svg.selectAll("circle.end-marker")
            .data(nodes.filter(n => n.isEndOfWord && n !== this.ac.root))
            .enter()
            .append("circle")
            .attr("class", "end-marker")
            .attr("cx", d => d.x + 12)
            .attr("cy", d => d.y - 12)
            .attr("r", 4)
            .attr("fill", textColor);
        
        // Add legend
        this.addLegend();
    },
    
    calculatePositions(nodes) {
        const levels = new Map();
        
        // Group nodes by depth
        nodes.forEach(node => {
            if (!levels.has(node.depth)) levels.set(node.depth, []);
            levels.get(node.depth).push(node);
        });
        
        const levelHeight = 70;
        const startY = 40;
        
        for (let [depth, levelNodes] of levels.entries()) {
            const y = startY + depth * levelHeight;
            const totalWidth = this.width - 100;
            
            if (levelNodes.length === 1) {
                levelNodes[0].x = this.width / 2;
                levelNodes[0].y = y;
            } else {
                const spacing = totalWidth / (levelNodes.length + 1);
                levelNodes.forEach((node, index) => {
                    node.x = 50 + spacing * (index + 1);
                    node.y = y;
                });
            }
        }
    },
    
    addLegend() {
        // Get current theme colors
        const computedStyle = getComputedStyle(document.documentElement);
        const textColor = computedStyle.getPropertyValue('--text-primary').trim() || '#1f2937';
        const secondaryTextColor = computedStyle.getPropertyValue('--text-secondary').trim() || '#6b7280';
        const bgColor = computedStyle.getPropertyValue('--bg-primary').trim() || '#ffffff';
        
        // Detect if we're in light mode
        const isLight = this.isLightTheme(bgColor, textColor);
        
        const legendData = [
            { 
                color: isLight ? "#ffffff" : "#374151", 
                stroke: textColor, 
                text: "State", 
                x: 20, 
                y: this.height - 80 
            },
            { 
                color: isLight ? "#ffffff" : "#374151", 
                stroke: textColor, 
                text: "Accept State", 
                x: 20, 
                y: this.height - 60, 
                marker: true 
            },
            { 
                color: isLight ? "#d1fae5" : "#065f46", 
                stroke: isLight ? "#16a34a" : "#10b981", 
                text: "Current Path", 
                x: 20, 
                y: this.height - 40 
            },
            { 
                color: "#22c55e", 
                stroke: "#16a34a", 
                text: "Match Found", 
                x: 20, 
                y: this.height - 20 
            }
        ];
        
        legendData.forEach(item => {
            this.svg.append("circle")
                .attr("cx", item.x + 10)
                .attr("cy", item.y)
                .attr("r", 8)
                .attr("fill", item.color)
                .attr("stroke", item.stroke)
                .attr("stroke-width", 1.5);
            
            if (item.marker) {
                this.svg.append("circle")
                    .attr("cx", item.x + 16)
                    .attr("cy", item.y - 6)
                    .attr("r", 3)
                    .attr("fill", textColor);
            }
            
            this.svg.append("text")
                .attr("x", item.x + 25)
                .attr("y", item.y + 4)
                .attr("font-size", "12px")
                .attr("fill", secondaryTextColor)
                .text(item.text);
        });
        
        // Failure link legend
        this.svg.append("line")
            .attr("x1", 200)
            .attr("y1", this.height - 60)
            .attr("x2", 230)
            .attr("y2", this.height - 60)
            .attr("stroke", "#ef4444")
            .attr("stroke-width", 1.5)
            .attr("stroke-dasharray", "5,5");
        
        this.svg.append("text")
            .attr("x", 240)
            .attr("y", this.height - 56)
            .attr("font-size", "12px")
            .attr("fill", secondaryTextColor)
            .text("Failure Link");
    },
    
    isLightTheme(bgColor, textColor) {
        if (!bgColor || !textColor) return true;
        
        const getBrightness = (color) => {
            if (color.startsWith('#')) {
                const hex = color.slice(1);
                const r = parseInt(hex.substr(0, 2), 16);
                const g = parseInt(hex.substr(2, 2), 16);
                const b = parseInt(hex.substr(4, 2), 16);
                return (r * 299 + g * 587 + b * 114) / 1000;
            }
            return bgColor.includes('white') || bgColor.includes('#fff') || 
                   bgColor.includes('#f') || textColor.includes('black') || 
                   textColor.includes('#000') || textColor.includes('#1') ? 255 : 50;
        };
        
        const bgBrightness = getBrightness(bgColor);
        return bgBrightness > 128;
    },
    
    search(text) {
        if (this.stepMode) {
            this.startStepMode(text);
            return;
        }
        
        const matches = this.ac.search(text);
        this.displayMatches(matches);
        this.visualizeText(text, matches);
        this.render(); // Re-render to show final state
        return matches;
    },
    
    startStepMode(text) {
        this.currentText = text;
        this.currentPosition = 0;
        this.ac.matches = [];
        this.ac.currentState = this.ac.root;
        this.stepMode = true;
        
        this.updateStatus(`Step mode: Processing character by character. Click "Step Through" to advance.`);
        this.render();
        this.visualizeText(text, []);
    },
    
    stepForward() {
        if (!this.stepMode || this.currentPosition >= this.currentText.length) {
            this.stepMode = false;
            this.updateStatus(`Step mode complete. Found ${this.ac.matches.length} matches.`);
            return;
        }
        
        const char = this.currentText[this.currentPosition];
        this.ac.processCharacter(char.toLowerCase(), this.currentPosition);
        
        // Update status with animation details
        let statusMsg = `Step ${this.currentPosition + 1}: Processing '${char}'`;
        if (this.ac.animationState.usedFailureLink) {
            statusMsg += ` (used failure link)`;
        }
        if (this.ac.animationState.newMatches.length > 0) {
            statusMsg += ` → Found: ${this.ac.animationState.newMatches.map(m => m.pattern).join(', ')}`;
        }
        this.updateStatus(statusMsg);
        
        this.currentPosition++;
        this.render();
        this.visualizeText(this.currentText, this.ac.matches);
    },
    
    resetAnimation() {
        this.stepMode = false;
        this.currentPosition = 0;
        this.ac.matches = [];
        this.ac.currentState = this.ac.root;
        this.ac.clearHighlights();
        this.render();
        this.updateStatus("Animation reset. Ready for new search.");
        document.getElementById('text-visualization').innerHTML = '';
    },
    
    displayMatches(matches) {
        const matchesEl = document.getElementById('ac-matches');
        if (matches.length === 0) {
            matchesEl.innerHTML = 'No matches found';
            matchesEl.style.color = '#6b7280';
        } else {
            matchesEl.innerHTML = `Found ${matches.length} match${matches.length > 1 ? 'es' : ''}: ${matches.map(m => `"${m.pattern}"`).join(', ')}`;
            matchesEl.style.color = '#22c55e';
        }
    },
    
    visualizeText(text, matches) {
        const textViz = document.getElementById('text-visualization');
        let html = '';
        
        for (let i = 0; i < text.length; i++) {
            let char = text[i];
            let inMatch = matches.some(m => i >= m.position && i <= m.endPosition);
            
            if (inMatch) {
                html += `<span class="matched-char">${char}</span>`;
            } else {
                html += `<span class="normal-char">${char}</span>`;
            }
        }
        
        textViz.innerHTML = html;
    },
    
    updateStatus(message) {
        document.getElementById('ac-status').textContent = message;
    }
};
 
// Initialize visualization
document.addEventListener('DOMContentLoaded', function() {
    ahoCorasickViz.init();
    
    // Pattern buttons
    document.querySelectorAll('.pattern-btn').forEach(btn => {
        btn.addEventListener('click', function() {
            const patterns = this.dataset.patterns.split(',').map(p => p.trim());
            ahoCorasickViz.setPatterns(patterns);
        });
    });
    
    // Search button
    document.getElementById('ac-search-btn').addEventListener('click', function() {
        const text = document.getElementById('ac-text-input').value;
        if (text && ahoCorasickViz.ac.patterns.length > 0) {
            ahoCorasickViz.stepMode = false;
            ahoCorasickViz.search(text);
        }
    });
    
    // Step through button
    document.getElementById('ac-step-btn').addEventListener('click', function() {
        const text = document.getElementById('ac-text-input').value;
        if (text && ahoCorasickViz.ac.patterns.length > 0) {
            if (!ahoCorasickViz.stepMode) {
                ahoCorasickViz.stepMode = true;
                ahoCorasickViz.startStepMode(text);
            } else {
                ahoCorasickViz.stepForward();
            }
        }
    });
    
    // Reset button
    document.getElementById('ac-reset-btn').addEventListener('click', function() {
        ahoCorasickViz.resetAnimation();
    });
    
    // Enter key support
    document.getElementById('ac-text-input').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') {
            document.getElementById('ac-search-btn').click();
        }
    });
    
    // Auto-search on input (only when not in step mode)
    document.getElementById('ac-text-input').addEventListener('input', function() {
        const text = this.value;
        if (text && ahoCorasickViz.ac.patterns.length > 0 && !ahoCorasickViz.stepMode) {
            ahoCorasickViz.search(text);
        }
    });
});
</script>
 
<style>
.aho-corasick-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-primary);
}

.ac-controls {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-bottom: 2rem;
}

.input-section, .pattern-section, .status-section {
    display: flex;
    gap: 1rem;
    align-items: center;
    flex-wrap: wrap;
}

.ac-controls input[type="text"] {
    padding: 0.5rem 1rem;
    border: 2px solid var(--border-primary);
    border-radius: 6px;
    background: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    min-width: 300px;
    flex: 1;
}

.ac-controls input[type="text"]:focus {
    outline: none;
    border-color: var(--accent-primary);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.ac-controls button {
    padding: 0.5rem 1rem;
    background: transparent;
    color: var(--text-primary);
    border: 1.5px solid var(--text-primary);
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    font-size: 14px;
    transition: all 0.2s ease;
    white-space: nowrap;
}

.ac-controls button:hover {
    background: var(--text-primary);
    color: var(--bg-primary);
    transform: translateY(-1px);
}

.pattern-btn {
    font-size: 12px !important;
    padding: 0.4rem 0.8rem !important;
}

#ac-reset-btn {
    border-color: #dc2626;
    color: #dc2626;
}

#ac-reset-btn:hover {
    background: #dc2626;
    color: #ffffff;
}

#aho-corasick-visualization {
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
    margin-bottom: 1rem;
    padding: 1rem;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 400px;
    overflow-x: auto;
}

.text-display {
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
}

#text-visualization {
    font-family: 'Courier New', monospace;
    font-size: 16px;
    line-height: 1.6;
    word-break: break-all;
    color: var(--text-primary);
}

.matched-char {
    background: #22c55e;
    color: white;
    padding: 2px 4px;
    border-radius: 3px;
    font-weight: bold;
}

.normal-char {
    color: var(--text-primary);
}

.status-section {
    font-size: 14px;
    font-weight: 500;
}

#ac-status {
    color: var(--text-secondary);
}

#ac-matches {
    color: var(--text-primary);
}

/* Mobile responsiveness for Aho-Corasick demo */
@media (max-width: 768px) {
    .aho-corasick-demo {
        padding: 1rem;
        margin: 1.5rem 0;
    }
    
    .input-section, .pattern-section, .status-section {
        flex-direction: column;
        align-items: stretch;
        gap: 0.75rem;
    }
    
    .ac-controls input[type="text"] {
        min-width: 100%;
        font-size: 16px; /* Prevents zoom on iOS */
    }
    
    .ac-controls button {
        width: 100%;
        justify-content: center;
    }
    
    .pattern-btn {
        font-size: 11px !important;
        padding: 0.5rem !important;
    }
    
    #aho-corasick-visualization {
        min-height: 300px;
        padding: 0.5rem;
    }
    
    #text-visualization {
        font-size: 14px;
    }
    
    .text-display {
        padding: 0.75rem;
    }
}

@media (max-width: 480px) {
    .aho-corasick-demo {
        padding: 0.75rem;
    }
    
    #aho-corasick-visualization {
        min-height: 250px;
    }
    
    .pattern-btn {
        font-size: 10px !important;
    }
    
    #text-visualization {
        font-size: 12px;
    }
}
</style>
 
Try the step-through mode above — it's actually pretty wild to watch. Click on "TCG, ATCG (overlapping!)" and then step through "ATCGATCG". You'll see something happen when the algorithm processes "ATCG" and then hits that final "A". Instead of giving up, it uses a failure link to jump to the "TCG" part of the trie and immediately finds another match!

This is the magic moment where Aho-Corasick shows its true power. A basic trie would have found "ATCG" at position 1 and then started over from position 2, completely missing that "TCG" was hiding at position 2. But with failure links, we catch both patterns in a single sweep.

So what just happened here? The Aho-Corasick automaton solved the fundamental problem with our trie approach. Instead of restarting from scratch every time we hit a dead end, those failure links let us maintain all the progress we've made.

When we're processing "ATCG" and looking for both "TCG" and "ATCG", the failure links ensure we're simultaneously tracking both possibilities. No more missed overlaps, no more multiple passes through the same DNA sequence.
 
The complexity ends up being **O(N + M + Z)** where N is text length, M is total pattern length, and Z is the number of matches. Compare that to:
- Our original nightmare: O(N × M × K) = 1000 trillion operations  
- Aho-Corasick: O(N + M + Z) ≈ 1 billion operations
 
Whether you've got 10 patterns or 10,000 patterns (like in our case), it still processes each character exactly once. That's the kind of scaling that makes you feel good about your algorithmic life choices.
 
So now we can blast through our billion-character DNA sequence, finding all 10,000 patterns in a single pass, catching every overlap and never missing a thing. We've gone from 116 days of computation to something that finishes in under a minute. Pretty solid improvement from where we started.
 
But here's the thing — we still haven't tackled our second problem. We can now find exact matches but what about that 100,000-character Oxford Nanopore read that might have 2-3% errors? How do we find "almost matches" when the DNA might have some mutations or sequencing errors?


## The Fuzzy Matching Problem (Or: When Life Gives You Noisy Data)

So it looks like we're shit out of luck with this next problem. We can't use Aho-Corasick since that's designed for exact matches, and here we are trying to find patterns that are "close enough" but not perfect(not to mention we only have one big pattern).

It seems like all we can do is fall back to our good old sliding window approach: align our pattern at every position, count how many characters match, and see if that's in the acceptable range. Maybe there exists a way to optimize this shit, but let's start simple.

Let me picture this on a smaller scale to wrap my head around it. Let's say we have:

**Text:** `ACGTAACGTAACGA` (this represents our billion-character long string)  
**Pattern:** `CGT` (this represents our 100,000-character pattern, but smaller for sanity)

If we're accepting 1 character mismatch (so 2 out of 3 characters need to match), let's slide our pattern across and count matches at each position:

```
Position 0: ACG vs CGT → 1 match  (C matches C)
Position 1: CGT vs CGT → 3 matches (perfect match!)
Position 2: GTA vs CGT → 1 match  (G matches G)
Position 3: TAA vs CGT → 0 matches
Position 4: AAC vs CGT → 1 match  (C matches C)
Position 5: ACG vs CGT → 1 match  (C matches C)
Position 6: CGT vs CGT → 3 matches (perfect match!)
Position 7: GTA vs CGT → 1 match  (G matches G)
Position 8: TAA vs CGT → 0 matches
Position 9: AAC vs CGT → 1 match  (C matches C)
Position 10: ACG vs CGT → 1 match (C matches C)
Position 11: CGA vs CGT → 2 matches (C and G match!)
```

So our match vector looks like: `[1, 3, 1, 0, 1, 1, 3, 1, 0, 1, 1, 2]`

According to this vector, the positions with values ≥2 are acceptable matches: **positions 1, 6, and 11**. Not bad!

Okay, so we have this process, and we would like to optimize it. Let's try to represent it mathematically—it's always a good practice to bring math into shit because mathematicians have worked really hard to optimize a lot of stuff, and maybe they've already solved our problem without realizing it.


## Converting Character Comparisons to Mathematical Operations

Soooo how can we represent this process mathematically? We need a way to convert the `==` check into some sort of mathematical shit, so that we can write formulas and stuff with this.

So here's an idea.

Let's say we have the text `ACCGTTACGGATTACGA` and a pattern `CGG`. What we can do is take one character at a time and encode it. If the position has that character, we write 1 there; if not, we write 0.

Let's take character `C`. For our text, `C` is present at positions 2, 3, 8, and 14 (using 1-based indexing):

**Text encoding for C:** `Tc = [0, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0]`

And for our pattern `CGG`, we have `C` at only the first position:

**Pattern encoding for C:** `Pc = [1, 0, 0]`

Now here's the things, we can do a sliding window approach, but on these encoded binary signals! We slide our pattern across the text, and wherever we see two `1`s aligning (both text and pattern have `1` at the same position), that's a character match. Then we just add up all the aligned `1`s to get the total matches at that position.

Let's do this for character `G` to make it clearer. For our text `ACCGTTACGGATTACGA`, `G` appears at positions 4, 9, 10, and 17:

**Text encoding for G:** `Tg = [0, 0, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1]`

And for our pattern `CGG`, we have `G` at positions 2 and 3:

**Pattern encoding for G:** `Pg = [0, 1, 1]`

Now, let me show you what happens when we align our pattern at different positions. When we align at position 3 (comparing text positions 3,4,5 with pattern positions 1,2,3):

- Text segment for G: `[0, 1, 0]` (positions 3,4,5 from Tg)
- Pattern for G: `[0, 1, 1]`
- Element-wise multiplication: `[0×0, 1×1, 0×1] = [0, 1, 0]`
- Sum: `0 + 1 + 0 = 1`

So at position 3, we have 1 matching `G` character. Let me do this for all positions to show you the pattern:

**Position 1:** Text segment `[0, 0, 0]` • Pattern `[0, 1, 1]` = 0  
**Position 2:** Text segment `[0, 0, 1]` • Pattern `[0, 1, 1]` = 1  
**Position 3:** Text segment `[0, 1, 0]` • Pattern `[0, 1, 1]` = 1  
**Position 4:** Text segment `[1, 0, 0]` • Pattern `[0, 1, 1]` = 0  
**Position 5:** Text segment `[0, 0, 0]` • Pattern `[0, 1, 1]` = 0  
**Position 6:** Text segment `[0, 0, 0]` • Pattern `[0, 1, 1]` = 0  
**Position 7:** Text segment `[0, 0, 1]` • Pattern `[0, 1, 1]` = 1  
**Position 8:** Text segment `[0, 1, 1]` • Pattern `[0, 1, 1]` = 2 (both Gs match!)  
**Position 9:** Text segment `[1, 1, 0]` • Pattern `[0, 1, 1]` = 1  
...and so on.

Now here's the beautiful part: this process of matching and adding is exactly like calculating the dot product of that text segment with the pattern!

Let me now do the complete demo with both C and G for a few key positions to show you how this works:

**Position 8 (where pattern CGG aligns with text CGG):**
- For C: Text segment `[1, 0, 0]` • Pattern `[1, 0, 0]` = 1
- For G: Text segment `[0, 1, 1]` • Pattern `[0, 1, 1]` = 2
- **Total matches = 1 + 2 = 3** Perfect match!

**Position 2 (where pattern CGG aligns with text CCG):**
- For C: Text segment `[1, 1, 0]` • Pattern `[1, 0, 0]` = 1
- For G: Text segment `[0, 0, 1]` • Pattern `[0, 1, 1]` = 1
- **Total matches = 1 + 1 = 2** (Close, but not perfect)

So mathematically, the frequency of matches for character `c` when aligning the pattern with position `i` can be given by:

$$
\text{matches}_c(i) = \sum_{j=0}^{|P|-1} T_c[i+j] \times P_c[j]
$$

And the total number of matching characters at position `i` is simply:

$$
\text{total_matches}(i) = \sum_{c \in \{A,T,G,C\}} \sum_{j=0}^{|P|-1} T_c[i+j] \times P_c[j]
$$

Let me show you with a DNA example. We'll slide our pattern across the text and see what happens:
 
<div class="slide-multiply-demo">
    <div class="controls">
        <input type="text" id="slide-text" value="ATCGATCG" placeholder="Enter DNA text (A,T,G,C)">
        <input type="text" id="slide-pattern" value="TCG" placeholder="Enter DNA pattern">
        <button id="slide-animate-btn">Animate Sliding</button>
        <button id="slide-reset-btn">Reset</button>
        <label>Speed: <input type="range" id="slide-speed" min="1" max="10" value="5"></label>
    </div>
    <div id="slide-visualization"></div>
    <div id="slide-explanation"></div>
</div>
 
<script>
let slideMultiplyDemo = {
    text: 'ATCGATCG',
    pattern: 'TCG',
    currentPosition: 0,
    animationId: null,
    speed: 5,
    
    init() {
        document.getElementById('slide-animate-btn').addEventListener('click', () => {
            this.text = document.getElementById('slide-text').value || 'ATCGATCG';
            this.pattern = document.getElementById('slide-pattern').value || 'TCG';
            this.startAnimation();
        });
        
        document.getElementById('slide-reset-btn').addEventListener('click', () => {
            this.reset();
        });
        
        document.getElementById('slide-speed').addEventListener('input', (e) => {
            this.speed = parseInt(e.target.value);
        });
        
        // Update when inputs change
        document.getElementById('slide-text').addEventListener('input', (e) => {
            this.text = e.target.value || 'ATCGATCG';
            this.currentPosition = 0;
            if (this.animationId) {
                cancelAnimationFrame(this.animationId);
                this.animationId = null;
            }
            this.render();
        });
        
        document.getElementById('slide-pattern').addEventListener('input', (e) => {
            this.pattern = e.target.value || 'TCG';
            this.currentPosition = 0;
            if (this.animationId) {
                cancelAnimationFrame(this.animationId);
                this.animationId = null;
            }
            this.render();
        });
        
        // Initial render
        this.render();
    },
    
    render() {
        const viz = document.getElementById('slide-visualization');
        let html = '<div class="slide-display">';
        
        // Show the text with signals
        html += '<div class="text-signals">';
        html += '<h4>Text Signals:</h4>';
        
        // Create character signals for DNA bases
        const allChars = [...new Set((this.text + this.pattern).toUpperCase().split(''))];
        const uniqueChars = allChars.filter(c => ['A', 'T', 'G', 'C'].includes(c));
        const textSignals = {};
        uniqueChars.forEach(char => {
            textSignals[char] = this.text.toUpperCase().split('').map(c => c === char ? 1 : 0);
        });
        
        // Display text
        html += '<div class="text-display">';
        html += '<div class="position-row">';
        for (let i = 0; i < this.text.length; i++) {
            html += `<div class="position-num">${i}</div>`;
        }
        html += '</div>';
        html += '<div class="text-row">';
        for (let i = 0; i < this.text.length; i++) {
            const isInWindow = i >= this.currentPosition && i < this.currentPosition + this.pattern.length;
            html += `<div class="text-char ${isInWindow ? 'highlighted' : ''}">${this.text[i]}</div>`;
        }
        html += '</div>';
        html += '</div>';
        
        // Show signals for each character
        uniqueChars.forEach(char => {
            html += `<div class="signal-row">`;
            html += `<div class="char-label">'${char}':</div>`;
            html += `<div class="signal-values">`;
            textSignals[char].forEach((val, i) => {
                const isInWindow = i >= this.currentPosition && i < this.currentPosition + this.pattern.length;
                html += `<div class="signal-bit ${val ? 'active' : ''} ${isInWindow ? 'in-window' : ''}">${val}</div>`;
            });
            html += `</div>`;
            html += `</div>`;
        });
        html += '</div>';
        
        // Show pattern signals
        html += '<div class="pattern-signals">';
        html += '<h4>Pattern Signals (sliding at position ' + this.currentPosition + '):</h4>';
        
        const patternSignals = {};
        uniqueChars.forEach(char => {
            patternSignals[char] = this.pattern.toUpperCase().split('').map(c => c === char ? 1 : 0);
        });
        
        uniqueChars.forEach(char => {
            html += `<div class="signal-row">`;
            html += `<div class="char-label">'${char}':</div>`;
            html += `<div class="signal-values">`;
            
            // Add spacing to show where pattern is
            for (let i = 0; i < this.currentPosition; i++) {
                html += `<div class="signal-bit spacer"></div>`;
            }
            
            patternSignals[char].forEach((val, i) => {
                html += `<div class="signal-bit ${val ? 'active pattern-bit' : 'pattern-bit'}">${val}</div>`;
            });
            
            // Add remaining spacers to align with text length
            const remainingSpaces = this.text.length - this.currentPosition - this.pattern.length;
            for (let i = 0; i < remainingSpaces; i++) {
                html += `<div class="signal-bit spacer"></div>`;
            }
            
            html += `</div>`;
            html += `</div>`;
        });
        html += '</div>';
        
        // Calculate and show the multiplication result
        html += '<div class="multiplication-result">';
        html += '<h4>Multiplication at position ' + this.currentPosition + ':</h4>';
        
        let totalScore = 0;
        const calculations = [];
        
        if (this.currentPosition + this.pattern.length <= this.text.length) {
            for (let i = 0; i < this.pattern.length; i++) {
                const textChar = this.text[this.currentPosition + i].toUpperCase();
                const patternChar = this.pattern[i].toUpperCase();
                const match = textChar === patternChar ? 1 : 0;
                totalScore += match;
                calculations.push({
                    textChar,
                    patternChar,
                    match
                });
            }
        }
        
        html += '<div class="calculation-grid">';
        calculations.forEach((calc, i) => {
            html += `<div class="calc-item ${calc.match ? 'match' : 'no-match'}">`;
            html += `<div class="calc-chars">'${calc.textChar}' × '${calc.patternChar}'</div>`;
            html += `<div class="calc-result">= ${calc.match}</div>`;
            html += `</div>`;
        });
        html += '</div>';
        
        html += `<div class="total-score">Total Score: ${totalScore} / ${this.pattern.length}`;
        if (totalScore === this.pattern.length) {
            html += ' <span class="match-found">MATCH FOUND!</span>';
        }
        html += '</div>';
        
        html += '</div>';
        html += '</div>';
        
        viz.innerHTML = html;
        
        // Update explanation
        const explanation = document.getElementById('slide-explanation');
        if (this.currentPosition + this.pattern.length > this.text.length) {
            explanation.innerHTML = '<p>Pattern extends beyond text. Sliding complete!</p>';
        } else if (totalScore === this.pattern.length) {
            explanation.innerHTML = `<p><strong>Match found at position ${this.currentPosition}!</strong> All ${this.pattern.length} characters matched perfectly.</p>`;
        } else {
            explanation.innerHTML = `<p>At position ${this.currentPosition}: ${totalScore} out of ${this.pattern.length} characters match.</p>`;
        }
    },
    
    startAnimation() {
        if (this.animationId) {
            cancelAnimationFrame(this.animationId);
        }
        
        // Make sure we have the latest values from inputs
        this.text = document.getElementById('slide-text').value || 'ATCGATCG';
        this.pattern = document.getElementById('slide-pattern').value || 'TCG';
        this.currentPosition = 0;
        let frameCount = 0;
        const framesPerStep = 60 / this.speed;
        
        const animate = () => {
            frameCount++;
            
            if (frameCount >= framesPerStep) {
                frameCount = 0;
                
                // Check if we're at the last valid position before incrementing
                if (this.currentPosition >= this.text.length - this.pattern.length) {
                    this.currentPosition = 0; // Loop back to start
                } else {
                    this.currentPosition++;
                }
                
                this.render();
            }
            
            this.animationId = requestAnimationFrame(animate);
        };
        
        animate();
    },
    
    reset() {
        if (this.animationId) {
            cancelAnimationFrame(this.animationId);
            this.animationId = null;
        }
        this.currentPosition = 0;
        this.render();
    }
};
 
document.addEventListener('DOMContentLoaded', () => slideMultiplyDemo.init());
</script>
 
<style>
.slide-multiply-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-primary);
}

.slide-multiply-demo .controls {
    display: flex;
    gap: 1rem;
    align-items: center;
    margin-bottom: 2rem;
    flex-wrap: wrap;
}

.slide-multiply-demo input[type="text"] {
    padding: 0.5rem 1rem;
    border: 2px solid var(--border-primary);
    border-radius: 6px;
    font-size: 14px;
    background: var(--bg-primary);
    color: var(--text-primary);
    flex: 1;
    min-width: 150px;
}

.slide-multiply-demo input[type="text"]:focus {
    outline: none;
    border-color: var(--accent-primary);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.slide-multiply-demo button {
    padding: 0.5rem 1rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    transition: all 0.2s ease;
    white-space: nowrap;
}

.slide-multiply-demo button:hover {
    background: #2563eb;
    transform: translateY(-1px);
}

.text-display {
    margin: 1rem 0;
}

.position-row, .text-row {
    display: flex;
    gap: 4px;
    flex-wrap: wrap;
}

.position-num {
    width: 30px;
    text-align: center;
    font-size: 10px;
    color: var(--text-quaternary);
}

.text-char {
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    font-family: 'Courier New', monospace;
    font-weight: bold;
    transition: all 0.3s ease;
    color: var(--text-primary);
}

.text-char.highlighted {
    background: #fbbf24;
    border-color: #f59e0b;
    transform: scale(1.1);
    color: #000;
}

.signal-row {
    display: flex;
    align-items: center;
    margin: 0.5rem 0;
    flex-wrap: wrap;
}

.char-label {
    width: 40px;
    font-weight: bold;
    font-family: 'Courier New', monospace;
    color: var(--text-primary);
    flex-shrink: 0;
}

.signal-values {
    display: flex;
    gap: 4px;
    flex-wrap: wrap;
}

.signal-bit {
    width: 30px;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 2px;
    font-size: 12px;
    font-weight: bold;
    transition: all 0.3s ease;
    color: var(--text-primary);
}

.signal-bit.active {
    background: #3b82f6;
    color: white;
}

.signal-bit.in-window {
    border-color: #f59e0b;
    border-width: 2px;
}

.signal-bit.pattern-bit {
    background: var(--bg-tertiary);
    border-color: #ef4444;
}

.signal-bit.pattern-bit.active {
    background: #ef4444;
    color: white;
}

.signal-bit.spacer {
    opacity: 0;
    pointer-events: none;
}

.text-signals, .pattern-signals, .multiplication-result {
    margin: 1.5rem 0;
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
}

.calculation-grid {
    display: flex;
    gap: 1rem;
    margin: 1rem 0;
    flex-wrap: wrap;
    justify-content: center;
}

/* FORCE WHITE BACKGROUNDS FOR CALCULATION ITEMS */
.slide-multiply-demo .calc-item,
.slide-multiply-demo .calc-item.match,
.slide-multiply-demo .calc-item.no-match,
.calculation-grid .calc-item,
.calculation-grid .calc-item.match,
.calculation-grid .calc-item.no-match {
    background-color: #ffffff !important;
    background: #ffffff !important;
    border: 2px solid #e5e7eb !important;
    color: #374151 !important;
    padding: 0.5rem !important;
    border-radius: 6px !important;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1) !important;
}

.slide-multiply-demo .calc-item.match,
.calculation-grid .calc-item.match {
    color: #059669 !important;
    border-color: #d1fae5 !important;
}

.slide-multiply-demo .calc-item.no-match,
.calculation-grid .calc-item.no-match {
    color: #dc2626 !important;
    border-color: #fee2e2 !important;
}

/* Dark mode support for calculation items */
@media (prefers-color-scheme: dark) {
    .calc-item {
        background: #374151 !important;
        border-color: #6b7280 !important;
        color: #f9fafb !important;
    }
    
    .calc-item.match {
        background: #064e3b !important;
        border-color: #10b981 !important;
        color: #a7f3d0 !important;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3) !important;
    }
    
    .calc-item.no-match {
        background: #7f1d1d !important;
        border-color: #f87171 !important;
        color: #fecaca !important;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3) !important;
    }
}

/* Additional fallback for theme-aware systems */
[data-theme="dark"] .calc-item,
.dark .calc-item {
    background: #374151 !important;
    border-color: #6b7280 !important;
    color: #f9fafb !important;
}

[data-theme="dark"] .calc-item.match,
.dark .calc-item.match {
    background: #064e3b !important;
    border-color: #10b981 !important;
    color: #a7f3d0 !important;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3) !important;
}

[data-theme="dark"] .calc-item.no-match,
.dark .calc-item.no-match {
    background: #7f1d1d !important;
    border-color: #f87171 !important;
    color: #fecaca !important;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3) !important;
}

.calc-chars {
    font-family: 'Courier New', monospace;
    font-weight: bold;
    text-align: center;
    color: inherit !important;
}

.calc-result {
    text-align: center;
    font-weight: bold;
    margin-top: 0.25rem;
    color: inherit !important;
}

.total-score {
    font-size: 18px;
    font-weight: bold;
    text-align: center;
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    color: var(--text-primary);
}

.match-found {
    color: #22c55e;
    font-size: 20px;
    animation: pulse 1s infinite;
}

@keyframes pulse {
    0% { opacity: 1; }
    50% { opacity: 0.7; }
    100% { opacity: 1; }
}

#slide-explanation {
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
    text-align: center;
    color: var(--text-primary);
}

/* Math theorem styling with dark mode support */
.math-theorem {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-tertiary);
    border-left: 4px solid #3b82f6;
    border-radius: 8px;
}

.math-theorem h4 {
    margin: 0 0 1rem 0;
    color: var(--accent-primary);
    font-size: 1.2rem;
}

.math-theorem p {
    margin: 0.5rem 0;
    color: var(--text-primary);
}

.math-theorem em {
    font-family: 'Times New Roman', serif;
    font-size: 1.1em;
    color: var(--accent-primary);
}

.math-details {
    margin-top: 1rem;
    padding: 1.5rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
    font-size: 0.95em;
    color: var(--text-primary);
}

.math-details code {
    background: var(--bg-primary);
    padding: 0.2rem 0.4rem;
    border-radius: 4px;
    color: var(--text-primary);
}

details summary {
    cursor: pointer;
    padding: 0.75rem 1rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    border: 1px solid var(--border-primary);
    font-weight: 600;
    transition: all 0.2s ease;
    color: var(--text-primary);
}

details summary:hover {
    background: var(--bg-primary);
    border-color: var(--accent-primary);
}

details[open] summary {
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
}

/* Mobile responsiveness for slide demo */
@media (max-width: 768px) {
    .slide-multiply-demo {
        padding: 1rem;
        margin: 1.5rem 0;
    }
    
    .slide-multiply-demo .controls {
        flex-direction: column;
        align-items: stretch;
        gap: 0.75rem;
    }
    
    .slide-multiply-demo input[type="text"] {
        min-width: 100%;
        font-size: 16px; /* Prevents zoom on iOS */
    }
    
    .slide-multiply-demo button {
        width: 100%;
        justify-content: center;
    }
    
    .signal-row {
        flex-direction: column;
        align-items: flex-start;
        gap: 0.5rem;
    }
    
    .char-label {
        width: auto;
    }
    
    .calculation-grid {
        justify-content: flex-start;
        gap: 0.5rem;
    }
    
    .position-row, .text-row, .signal-values {
        flex-wrap: wrap;
        gap: 2px;
    }
    
    .position-num, .text-char, .signal-bit {
        width: 25px;
        height: 25px;
        font-size: 10px;
    }
    
    .total-score {
        font-size: 16px;
    }
}

@media (max-width: 480px) {
    .slide-multiply-demo {
        padding: 0.75rem;
    }
    
    .position-num, .text-char, .signal-bit {
        width: 20px;
        height: 20px;
        font-size: 9px;
    }
    
    .calc-item {
        padding: 0.25rem;
        font-size: 12px;
    }
    
    .total-score {
        font-size: 14px;
        padding: 0.75rem;
    }
}
</style>


## Wait, This Math Looks Familiar... 

So we've converted our sliding window into this fancy math formula, but now what? Last I checked, this will still be O(N×M) since we're just matching, adding, and sliding. In fact, it's a little worse since now we're processing each character individually.

And you're absolutely right—it is in fact worse if we continue doing the sliding window approach here. The math equation we derived for a single character:

$$
\text{matches}_c(i) = \sum_{j=0}^{|P|-1} T_c[i+j] \times P_c[j]
$$

This sliding-and-multiplying operation? It's almost like something called **convolution**...

In true convolution, we'd actually need to flip our pattern backwards first. So "TCG" would become "GCT", then we'd slide that across. But for pattern matching, we don't want the backwards version—we want to find "TCG", not "GCT"!

What we're actually doing is called **cross-correlation**. It's convolution's cooler cousin that doesn't flip things around.

For those who don't know what convolution is: it's a mathematical operation used heavily in signal processing to measure how much one signal "looks like" another signal at different time shifts. Think of it as asking "how similar are these two signals when I slide one across the other?" 

For continuous signals, the equation uses integration:

$$
(f * g)(t) = \int_{-\infty}^{\infty} f(\tau) \cdot g(t - \tau) \, d\tau
$$

But since we're dealing with discrete sequences (like our DNA strings), we use discrete convolution with summation instead:

$$
(f * g)[n] = \sum_{m=-\infty}^{\infty} f[m] \times g[n - m]
$$

Now, let me rewrite our sequence matching equation in a way that'll make your eyes light up. Look at that inner sum again:

$$
\sum_{j=0}^{|P|-1} T_c[i+j] \times P_c[j]
$$

This is **exactly** the definition of cross-correlation between $$T_c$$ and $$P_c$$ at position i! And cross-correlation is just convolution with one signal flipped. If we reverse our pattern $$P_c$$ to get $$P_c^{rev}$$, then we can write our operation as a convolution:

$$
\text{matches}_c(i) = (T_c * P_c^{rev})[i]
$$

Where `*` is the convolution operator! So our total matching becomes:

$$
\text{total_matches}(i) = \sum_{c \in \{A,T,G,C\}} (T_c * P_c^{rev})[i]
$$

We just wrote pattern matching as a convolution problem!

But wait, why did signals suddenly show up here? Weren't we dealing with text and DNA sequences, not signals? You're probably thinking: "Signals are for audio and radio waves and stuff. Text is just... text. Letters. Symbols. Not wavy things!"

But here's the thing: the moment we represented text as math using those 0/1 encodings, we converted it into signals!

For example, for a DNA sequence like `AGGCGTA`, we encode G as:
```
G: [0, 1, 1, 0, 1, 0, 0]
```

Now, if we treat the increasing index positions as the flow of time, we get ourselves nice signal-looking things. In fact, that's exactly what a signal is—a bunch of values changing over time.

We can even graph this like signals:

<div id="signal-graph-demo" style="margin: 2rem 0; padding: 2rem; background: var(--bg-secondary); border-radius: 8px; border: 1px solid var(--border-primary);">
    <div style="margin-bottom: 1rem;">
        <label style="color: var(--text-primary); font-weight: 600;">DNA Sequence: 
            <input type="text" id="dna-input" value="AGGCGTA" style="padding: 0.5rem; margin-left: 0.5rem; border: 1px solid var(--border-primary); border-radius: 4px; background: var(--bg-primary); color: var(--text-primary); font-size: 16px; width: 200px;">
        </label>
    </div>
    <div style="overflow-x: auto; border-radius: 6px; border: 1px solid var(--border-primary); background: var(--bg-primary);">
        <canvas id="signal-canvas" width="700" height="280" style="display: block; min-width: 600px;"></canvas>
    </div>
    <div style="margin-top: 1rem; font-size: 14px; color: var(--text-secondary); text-align: center;">
        Signal representation: 1 when we see G, 0 otherwise
    </div>
</div>

<style>
#signal-graph-demo input:focus {
    outline: none;
    border-color: var(--accent-primary);
}

@media (max-width: 768px) {
    #signal-graph-demo {
        padding: 1rem !important;
        margin: 1.5rem 0 !important;
    }
    
    #signal-graph-demo label {
        display: block !important;
        margin-bottom: 0.5rem !important;
    }
    
    #signal-graph-demo input {
        margin-left: 0 !important;
        width: 100% !important;
        max-width: 300px !important;
    }
    
    #signal-canvas {
        min-width: 500px !important;
    }
}

@media (max-width: 480px) {
    #signal-graph-demo {
        padding: 0.75rem !important;
    }
    
    #signal-canvas {
        min-width: 400px !important;
        height: 250px !important;
    }
}
</style>

<script>
function drawSignalGraph() {
    const canvas = document.getElementById('signal-canvas');
    const ctx = canvas.getContext('2d');
    const dnaSequence = document.getElementById('dna-input').value.toUpperCase() || 'AGGCGTA';
    
    // Set up high DPI rendering
    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
    ctx.scale(dpr, dpr);
    canvas.style.width = rect.width + 'px';
    canvas.style.height = rect.height + 'px';
    
    // Clear canvas
    ctx.clearRect(0, 0, rect.width, rect.height);
    
    // Get theme colors
    const computedStyle = getComputedStyle(document.documentElement);
    const bgColor = computedStyle.getPropertyValue('--bg-primary').trim() || '#ffffff';
    const textColor = computedStyle.getPropertyValue('--text-primary').trim() || '#1f2937';
    const secondaryTextColor = computedStyle.getPropertyValue('--text-secondary').trim() || '#6b7280';
    
    // Detect theme for better colors
    const isLightTheme = !bgColor.includes('rgb') ? 
        (bgColor === '#ffffff' || bgColor === 'white' || !bgColor.startsWith('#')) : 
        bgColor.match(/\d+/g)?.reduce((sum, val) => sum + parseInt(val), 0) > 384;
    
    // Theme-appropriate signal colors
    const signalColor = isLightTheme ? '#059669' : '#10b981';
    const axisColor = isLightTheme ? textColor : textColor;
    
    // Fill canvas background
    ctx.fillStyle = bgColor;
    ctx.fillRect(0, 0, rect.width, rect.height);
    
    // Signal data
    const signal = dnaSequence.split('').map(char => char === 'G' ? 1 : 0);
    
    const margin = 80;
    const plotWidth = rect.width - 2 * margin;
    const plotHeight = 120;
    const y = 60;
    
    // Title
    ctx.fillStyle = textColor;
    ctx.font = '18px -apple-system, BlinkMacSystemFont, sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(`Signal for G in "${dnaSequence}"`, rect.width/2, 30);
    
    // Draw axes
    ctx.strokeStyle = axisColor;
    ctx.lineWidth = 1;
    ctx.beginPath();
    // Y-axis
    ctx.moveTo(margin, y);
    ctx.lineTo(margin, y + plotHeight);
    // X-axis
    ctx.moveTo(margin, y + plotHeight);
    ctx.lineTo(margin + plotWidth, y + plotHeight);
    ctx.stroke();
    
    // Y-axis labels
    ctx.fillStyle = textColor;
    ctx.font = '14px -apple-system, BlinkMacSystemFont, sans-serif';
    ctx.textAlign = 'right';
    ctx.fillText('1', margin - 10, y + 15);
    ctx.fillText('0', margin - 10, y + plotHeight - 5);
    
    // Draw signal line
    ctx.strokeStyle = signalColor;
    ctx.lineWidth = 2;
    ctx.beginPath();
    
    signal.forEach((value, i) => {
        const x = margin + (i / Math.max(signal.length - 1, 1)) * plotWidth;
        const signalY = y + plotHeight - (value * plotHeight * 0.7) - 15;
        
        if (i === 0) {
            ctx.moveTo(x, signalY);
        } else {
            const prevX = margin + ((i - 1) / Math.max(signal.length - 1, 1)) * plotWidth;
            const prevValue = signal[i - 1];
            const prevSignalY = y + plotHeight - (prevValue * plotHeight * 0.7) - 15;
            
            ctx.lineTo(x, prevSignalY);
            ctx.lineTo(x, signalY);
        }
    });
    
    if (signal.length > 0) {
        const lastX = margin + plotWidth;
        const lastValue = signal[signal.length - 1];
        const lastSignalY = y + plotHeight - (lastValue * plotHeight * 0.7) - 15;
        ctx.lineTo(lastX, lastSignalY);
    }
    ctx.stroke();
    
    // Draw points and labels
    signal.forEach((value, i) => {
        const x = margin + (i / Math.max(signal.length - 1, 1)) * plotWidth;
        const signalY = y + plotHeight - (value * plotHeight * 0.7) - 15;
        
        // Draw point
        ctx.fillStyle = signalColor;
        ctx.beginPath();
        ctx.arc(x, signalY, 4, 0, 2 * Math.PI);
        ctx.fill();
        
        // Value label
        ctx.fillStyle = textColor;
        ctx.font = '12px -apple-system, BlinkMacSystemFont, sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText(value.toString(), x, signalY + (value === 1 ? -12 : 18));
        
        // Position number
        ctx.fillStyle = secondaryTextColor;
        ctx.font = '11px -apple-system, BlinkMacSystemFont, sans-serif';
        ctx.fillText(i.toString(), x, y + plotHeight + 15);
        
        // DNA base
        ctx.fillStyle = dnaSequence[i] === 'G' ? signalColor : textColor;
        ctx.font = '14px -apple-system, BlinkMacSystemFont, sans-serif';
        ctx.fillText(dnaSequence[i], x, y + plotHeight + 30);
    });
    
    // Position label
    ctx.fillStyle = secondaryTextColor;
    ctx.font = '12px -apple-system, BlinkMacSystemFont, sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('Position', rect.width/2, y + plotHeight + 50);
    
    // Explanation
    ctx.fillStyle = textColor;
    ctx.font = '13px -apple-system, BlinkMacSystemFont, sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('Signal = 1 when we see G, 0 otherwise', rect.width/2, y + plotHeight + 70);
}

// Initialize and add event listener
document.addEventListener('DOMContentLoaded', () => {
    drawSignalGraph();
    document.getElementById('dna-input').addEventListener('input', drawSignalGraph);
});
</script>

So text over position is analogous to signals over time. Now what?


## Enter the Frequency Domain (AKA "Let's Change Our Perspective")

Now that we know text can be treated as signals and we need to find convolution, we can use some tricks that mathematicians have already discovered for us over the centuries. 

Right now we have a text "signal" over "time" (position), so we're seeing our signal from the perspective of time. But what if we flip the script and look at this same signal from the perspective of **frequency**?

Now, I know what you're thinking: "Time being analogous to position was already a bit of a stretch, but *frequency* in text? That sounds completely bonkers!" And you're absolutely right—it does sound weird. You could think of it as the frequency of G appearing in the text, but honestly, that's not quite it either.

Here's the beautiful thing about math: **it doesn't care** if our signal is coming from text, audio, radio waves, or your heartbeat monitor. For math, this is just a signal over time, and math will happily transform it to see it from frequency's perspective. It's like putting on different glasses to see the same object from a completely different angle.

But wait—why do we even *want* to do this? You'll understand in a moment, just bear with me!

Math provides us with this magical tool called the **Discrete Fourier Transform (DFT)** which is defined as:

$$
X[k] = \sum_{n=0}^{N-1} x[n] \cdot e^{-j2\pi kn/N}
$$

This transforms our "time domain" signal to "frequency domain". It's math's way of looking at the exact same data from a completely different perspective—like viewing a sculpture from the side instead of the front.

If you don't understand how the transformation itself works, I'd love to explain that, but that's not our mission here. We're here to find patterns in strings (remember that? That's kinda why we started this whole journey!). So all you need to know is that this formula takes our position-based signal and gives us a frequency-based view of the same information.

"Okay, cool math trick," you might say, "but how does this help us find DNA patterns faster?"

Well, here comes the beautiful part! There's this absolutely gorgeous property about convolution that connects both domains. It's called the **Convolution Theorem**, and it says that the Fourier transform of a convolution in the time domain equals the element-wise multiplication of the Fourier transforms of the individual functions:

$$
\mathcal{F}\{f * g\} = \mathcal{F}\{f\} \odot \mathcal{F}\{g\}
$$

In plain English: convolution in time domain = element-wise multiplication in frequency domain.

This means instead of sliding and summing (expensive!), we can just multiply corresponding frequency components together (cheap!).

Which essentially means that if were to calculate this convolution in frequency domain it would mean only doing element wise multiplication which is actually just O(N) time.

And *that* is what we're going to use to make our pattern matching blazingly(sorta) fast!


### But Why Does This Work?
 
<details>
<summary><strong>Click for the FULL mathematical journey</strong></summary>
 
<div class="math-details">
 
<h4>Part 1: Setting Up Our Problem</h4>
 
Let's start with what we're actually trying to compute. For pattern matching, at each position $i$ in our text, we want to know how many characters match:
 
$$score[i] = \sum_{j=0}^{|pattern|-1} \mathbb{1}_{text[i+j] = pattern[j]}$$
 
Where $\mathbb{1}$ is the indicator function (1 if true, 0 if false).
 
But we can't FFT this directly because of that pesky equality check. So we use our character signal trick! For each character $c$ in our alphabet:
 
$$text_c[i] = \begin{cases} 1 & \text{if } text[i] = c \\ 0 & \text{otherwise} \end{cases}$$
 
$$pattern_c[j] = \begin{cases} 1 & \text{if } pattern[j] = c \\ 0 & \text{otherwise} \end{cases}$$
 
Now our score becomes:
 
$$score[i] = \sum_{c \in \Sigma} \sum_{j=0}^{|pattern|-1} text_c[i+j] \cdot pattern_c[j]$$
 
This inner sum looks familiar... it's a correlation!
 
<h4>Part 2: From Correlation to Convolution</h4>
 
The correlation of $text_c$ and $pattern_c$ at position $i$ is:
 
$$(text_c \star pattern_c)[i] = \sum_{j=0}^{|pattern|-1} text_c[i+j] \cdot pattern_c[j]$$
 
But FFT works with convolution, not correlation. The convolution is:
 
$$(text_c * pattern_c)[i] = \sum_{j=0}^{|pattern|-1} text_c[j] \cdot pattern_c[i-j]$$
 
Notice the difference? In convolution, one signal is flipped. But here's a trick: correlation is just convolution with one signal reversed!
 
$$(text_c \star pattern_c)[i] = (text_c * \overline{pattern_c})[i]$$
 
Where $\overline{pattern_c}[j] = pattern_c[|pattern|-1-j]$ (the reversed pattern).
 
<h4>Part 3: Enter the Discrete Fourier Transform</h4>
 
The DFT of a signal $x$ of length $N$ is:
 
$$X[k] = \sum_{n=0}^{N-1} x[n] \cdot e^{-i2\pi kn/N}$$
 
Let's break this down:
- $k$ is the frequency index (0 to N-1)
- $e^{-i2\pi kn/N}$ is a complex exponential that rotates $k$ times as $n$ goes from 0 to N-1
- We're decomposing our signal into N different frequencies
 
The inverse DFT is:
 
$$x[n] = \frac{1}{N} \sum_{k=0}^{N-1} X[k] \cdot e^{i2\pi kn/N}$$
 
<h4>Part 4: The Convolution Theorem (The Main Event!)</h4>
 
Here's where the magic happens. Let's prove that convolution in time domain equals element-wise multiplication in frequency domain.
 
Start with the convolution:
 
$$(f * g)[n] = \sum_{m=0}^{N-1} f[m] \cdot g[n-m]$$
 
Take the DFT of both sides:
 
$$\mathcal{F}\{(f * g)\}[k] = \sum_{n=0}^{N-1} \left(\sum_{m=0}^{N-1} f[m] \cdot g[n-m]\right) \cdot e^{-i2\pi kn/N}$$
 
Swap the sums (we can do this because they're finite):
 
$$= \sum_{m=0}^{N-1} f[m] \sum_{n=0}^{N-1} g[n-m] \cdot e^{-i2\pi kn/N}$$
 
Now here's the clever bit. Let $j = n - m$, so $n = j + m$. Note that as $n$ ranges from $0$ to $N-1$ and $m$ is fixed, $j$ also ranges from $-m$ to $N-1-m$. For circular convolution (which is what DFT computes), we use modular arithmetic, so this becomes:
 
$$= \sum_{m=0}^{N-1} f[m] \sum_{j=0}^{N-1} g[j] \cdot e^{-i2\pi k(j+m)/N}$$
 
Split the exponential:
 
$$= \sum_{m=0}^{N-1} f[m] \cdot e^{-i2\pi km/N} \sum_{j=0}^{N-1} g[j] \cdot e^{-i2\pi kj/N}$$
 
But wait! Those are just the DFTs of $f$ and $g$:
 
$$= \left(\sum_{m=0}^{N-1} f[m] \cdot e^{-i2\pi km/N}\right) \cdot \left(\sum_{j=0}^{N-1} g[j] \cdot e^{-i2\pi kj/N}\right)$$
 
$$= F[k] \cdot G[k]$$
 
**BOOM!** 🎆 We just proved the convolution theorem!
 
<h4>Part 5: Why This Is Fast (The Complexity Analysis)</h4>
 
Direct convolution:
- For each of N positions, sum over M pattern positions
- Complexity: $O(N \times M)$
 
FFT approach:
1. FFT of text signal: $O(N \log N)$
2. FFT of pattern signal: $O(N \log N)$ (padded to same length)
3. Element-wise multiplication: $O(N)$
4. Inverse FFT: $O(N \log N)$
- Total: $O(N \log N)$
 
The FFT algorithm achieves this speed by using a divide-and-conquer approach, recursively breaking down the DFT computation into smaller subproblems. The implementation details of FFT are beyond the scope of this article, but if you're curious about how it works under the hood, check out this excellent resource: [Fast Fourier Transform on CP-Algorithms](https://cp-algorithms.com/algebra/fft.html).
 
<h4>Part 6: Putting It All Together for Text Matching</h4>
 
For our text matching problem:
 
1. For each character $c$, create binary signals $text_c$ and $pattern_c$
2. Compute $score_c[i] = (text_c \star pattern_c)[i]$ using FFT:
   - $\mathcal{F}\{text_c\}$
   - $\mathcal{F}\{\overline{pattern_c}\}$ (or just conjugate $\mathcal{F}\{pattern_c\}$)
   - Multiply element-wise
   - Inverse FFT
3. Sum over all characters: $score[i] = \sum_{c \in \Sigma} score_c[i]$
4. Positions where $score[i] = |pattern|$ are exact matches!
 
The total complexity is $O(|\Sigma| \cdot N \log N)$ where $|\Sigma|$ is the alphabet size. Since the alphabet is fixed (e.g., 128 for ASCII), this is effectively $O(N \log N)$.
 
<h4>Bonus: The Complex Number Magic</h4>
 
Why do complex numbers make this work? The key is Euler's formula:
 
$$e^{i\theta} = \cos(\theta) + i\sin(\theta)$$
 
So our DFT basis functions $e^{-i2\pi kn/N}$ are actually spinning around the unit circle in the complex plane. Each frequency $k$ spins at a different rate, and the DFT measures how much our signal "resonates" with each spinning frequency.
 
When we multiply element-wise in the frequency domain, we're combining these resonances. The inverse FFT then reconstructs the convolution by adding up all these spinning components with the right phases.
 
It's like... imagine you're trying to predict where two spinning dancers will meet. Instead of tracking their every move (convolution), you can:
1. Figure out how fast each is spinning (FFT)
2. Multiply their spin rates element-wise (frequency domain multiplication)
3. Calculate where they'll meet (inverse FFT)
 
That's the essence of why this beautiful theorem works!
 
</div>
</details>


## The Magic Recipe: FFT-Based Pattern Matching

Alright, let's put all the pieces together and see how this frequency domain magic actually solves our original problem.

Remember, we need to find the convolution to get the number of matches at each position. Here's our game plan:

1. **Transform text signal to frequency domain** using Fourier Transform
2. **Transform pattern signal to frequency domain** using Fourier Transform  
3. **Multiply them element wise** (this is where the convolution theorem shines!)
4. **Transform back to time domain** using Inverse Fourier Transform

And boom! We get a vector containing the number of character matches at each position.

Let me break this down step by step:

**Step 1 & 2: Into the Frequency Realm**
We take our text signal (like that G signal we saw earlier) and our pattern signal, and transform both using the DFT formula we saw. This converts our position-based signals into frequency-based representations.

**Step 3: The Multiplication Magic**
Instead of doing the expensive convolution in the time domain (which would be O(N×M)), we just multiply the transformed signals element-wise. This is lightning fast—just O(N) operations! The convolution theorem guarantees this gives us exactly what we want.

**Step 4: Back to Reality**
We use the **Inverse Fourier Transform** to convert our result back to the time domain. As the name suggests, it's the reverse operation that brings us back to our familiar position-based world.

The result? A vector where each position tells us exactly how many characters matched when we aligned our pattern at that position! Whether we want 2% error tolerance or 3% error tolerance doesn't matter—we have the exact match count for every position, so we can easily apply any threshold we want.

Seems almost magical, doesn't it? Like we cheated the universe by taking a detour through frequency space!

Now how much time does this whole process actually take?

To calculate the Fourier transform, we use an algorithm called **FFT (Fast Fourier Transform)**. This beautiful algorithm calculates the Fourier transform in O(N log N) time using a divide-and-conquer approach. If you're curious about the nitty-gritty details of how FFT works its magic, you can dive deeper at [cp-algorithms.com](https://cp-algorithms.com/algebra/fft.html).

Let's break down the time complexity:
- **FFT on text**: O(N log N)
- **FFT on pattern**: O(M log M) ≈ O(N log N) (since we pad to same size)
- **Element-wise multiplication**: O(N) (super fast!)
- **Inverse FFT**: O(N log N)

**Total time complexity: O(N log N)**

That's a *massive* improvement over our original O(N×M) brute force approach!

Remember our second bioinformatics problem? We had a 100,000-base pattern to search in a 1-billion-character genome. 

**Before (brute force):** 10^9 × 10^5 = 10^14 operations
- At 10^8 operations per second: 10^14 / 10^8 = 10^6 seconds = **11.6 days**

**After (FFT):** ~10^9 × log(10^9) ≈ 10^9 × 30 ≈ 3×10^10 operations
- At 10^8 operations per second: 3×10^10 / 10^8 = 300 seconds = **5 minutes**

From **11.6 days** to **5 minutes**. That's not just an improvement—that's a complete game changer! We went from "let's schedule this for next week" to "grab a coffee while it runs."

And the best part? This works for *any* error tolerance. Want exact matches? Check positions with full match count. Want 2% tolerance? Check positions with ≥98% match count. Want 5% tolerance? Check positions with ≥95% match count. Same algorithm, different thresholds!


## The Complete FFT Pattern Matching Demo

Alright, enough theory! Let's see this magic in action. This interactive demo will walk you through every single step of the FFT-based pattern matching algorithm. You'll see exactly how we transform text into the frequency domain, multiply signals, and transform back to get our match scores.

<div class="fft-demo-container">
    <div class="demo-controls">
        <div class="input-group">
            <label for="fft-text">Text (DNA sequence):</label>
            <input type="text" id="fft-text" value="ATCGATCGATCG" placeholder="Enter DNA text (A,T,G,C)" maxlength="12">
        </div>
        <div class="input-group">
            <label for="fft-pattern">Pattern:</label>
            <input type="text" id="fft-pattern" value="TCGA" placeholder="Enter pattern" maxlength="6">
        </div>
        <div class="demo-buttons">
            <button id="fft-run-demo">Run Complete Demo</button>
            <button id="fft-step-demo">Step Through</button>
            <button id="fft-reset-demo">Reset</button>
        </div>
    </div>

    <div class="demo-steps">
        <!-- Step 1: Input Processing -->
        <div class="demo-step" id="step-input">
            <h3>Step 1: Input Processing</h3>
            <div class="step-content">
                <div class="input-display">
                    <div class="text-row">
                        <span class="label">Text:</span>
                        <div class="sequence-display" id="text-display"></div>
                    </div>
                    <div class="pattern-row">
                        <span class="label">Pattern:</span>
                        <div class="sequence-display" id="pattern-display"></div>
                    </div>
                </div>
                <div class="step-explanation" id="input-explanation">
                    Enter your DNA text and pattern above. We'll convert each character into binary signals for A, T, G, and C.
                </div>
            </div>
        </div>

        <!-- Step 2: Signal Conversion -->
        <div class="demo-step" id="step-signals">
            <h3>Step 2: Converting to Binary Signals</h3>
            <div class="step-content">
                <div class="signals-grid" id="signals-grid"></div>
                <div class="step-explanation" id="signals-explanation">
                    Each character type gets its own binary signal. A '1' means the character is present at that position, '0' means it's not.
                </div>
            </div>
        </div>

        <!-- Step 3: FFT of Text -->
        <div class="demo-step" id="step-text-fft">
            <h3>Step 3: FFT of Text Signals</h3>
            <div class="step-content">
                <div class="fft-visualization" id="text-fft-viz"></div>
                <div class="step-explanation" id="text-fft-explanation">
                    We transform each text signal from time domain to frequency domain using FFT. The complex numbers represent different frequency components.
                </div>
            </div>
        </div>

        <!-- Step 4: FFT of Pattern -->
        <div class="demo-step" id="step-pattern-fft">
            <h3>Step 4: FFT of Pattern Signals (Reversed)</h3>
            <div class="step-content">
                <div class="fft-visualization" id="pattern-fft-viz"></div>
                <div class="step-explanation" id="pattern-fft-explanation">
                    We reverse the pattern signals and transform them to frequency domain. This gives us the correlation instead of convolution.
                </div>
            </div>
        </div>

        <!-- Step 5: Multiplication -->
        <div class="demo-step" id="step-multiply">
            <h3>Step 5: Frequency Domain Multiplication</h3>
            <div class="step-content">
                <div class="multiplication-grid" id="multiply-grid"></div>
                <div class="step-explanation" id="multiply-explanation">
                    In frequency domain, convolution becomes simple element-wise multiplication! This is where the magic happens.
                </div>
            </div>
        </div>

        <!-- Step 6: IFFT -->
        <div class="demo-step" id="step-ifft">
            <h3>Step 6: Inverse FFT Back to Time Domain</h3>
            <div class="step-content">
                <div class="ifft-visualization" id="ifft-viz"></div>
                <div class="step-explanation" id="ifft-explanation">
                    We transform the multiplied signals back to time domain using inverse FFT. This gives us the correlation for each character.
                </div>
            </div>
        </div>

        <!-- Step 7: Final Results -->
        <div class="demo-step" id="step-results">
            <h3>Step 7: Final Pattern Matching Results</h3>
            <div class="step-content">
                <div class="results-display" id="results-display"></div>
                <div class="step-explanation" id="results-explanation">
                    We sum up the correlations for all characters to get the final match scores at each position!
                </div>
            </div>
        </div>
    </div>
    <div class="demo-progress">
        <div class="progress-bar">
            <div class="progress-fill" id="progress-fill"></div>
        </div>
        <div class="progress-text" id="progress-text">Ready to start</div>
    </div>
</div>

<style>
.fft-demo-container {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-primary);
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.demo-controls {
    background: var(--bg-primary);
    padding: 1.5rem;
    border-radius: 8px;
    margin-bottom: 2rem;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    border: 1px solid var(--border-primary);
}

.input-group {
    margin-bottom: 1rem;
}

.input-group label {
    display: block;
    font-weight: 600;
    margin-bottom: 0.5rem;
    color: var(--text-primary);
}

.input-group input {
    width: 100%;
    padding: 0.75rem;
    border: 2px solid var(--border-primary);
    border-radius: 6px;
    font-size: 16px;
    font-family: 'Courier New', monospace;
    letter-spacing: 2px;
    text-transform: uppercase;
    background: var(--bg-primary);
    color: var(--text-primary);
}

.input-group input:focus {
    outline: none;
    border-color: var(--accent-primary);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.demo-buttons {
    display: flex;
    gap: 1rem;
    margin-top: 1.5rem;
    flex-wrap: wrap;
}

.demo-buttons button {
    padding: 0.75rem 1.5rem;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
}

#fft-run-demo {
    background: #10b981;
    color: white;
}

#fft-run-demo:hover {
    background: #059669;
    transform: translateY(-1px);
}

#fft-step-demo {
    background: #3b82f6;
    color: white;
}

#fft-step-demo:hover {
    background: #2563eb;
    transform: translateY(-1px);
}

#fft-reset-demo {
    background: #6b7280;
    color: white;
}

#fft-reset-demo:hover {
    background: #4b5563;
    transform: translateY(-1px);
}

.demo-steps {
    margin-bottom: 2rem;
}

.demo-step {
    background: var(--bg-primary);
    margin-bottom: 1.5rem;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    opacity: 0.3;
    transition: all 0.3s ease;
    border: 1px solid var(--border-primary);
}

.demo-step.active {
    opacity: 1;
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
    border: 2px solid var(--accent-primary);
}

.demo-step h3 {
    background: var(--bg-secondary);
    margin: 0;
    padding: 1rem 1.5rem;
    color: var(--text-primary);
    border-bottom: 1px solid var(--border-primary);
}

.demo-step.active h3 {
    background: var(--accent-primary);
    color: white;
}

.step-content {
    padding: 1.5rem;
}

.input-display {
    margin-bottom: 1rem;
}

.text-row, .pattern-row {
    display: flex;
    align-items: center;
    margin-bottom: 0.5rem;
}

.label {
    min-width: 80px;
    font-weight: 600;
    color: var(--text-primary);
}

.sequence-display {
    display: flex;
    gap: 2px;
    font-family: 'Courier New', monospace;
    font-size: 18px;
    font-weight: bold;
}

.sequence-char {
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 2px solid var(--border-primary);
    border-radius: 4px;
    background: var(--bg-primary);
}

.sequence-char.A { background: #fee2e2; border-color: #fca5a5; color: #dc2626; }
.sequence-char.T { background: #dbeafe; border-color: #93c5fd; color: #2563eb; }
.sequence-char.G { background: #dcfce7; border-color: #86efac; color: #16a34a; }
.sequence-char.C { background: #fef3c7; border-color: #fcd34d; color: #d97706; }

.signals-grid {
    display: grid;
    gap: 1rem;
    margin-bottom: 1rem;
}

.signal-row {
    display: flex;
    align-items: center;
    gap: 1rem;
}

.signal-label {
    min-width: 40px;
    font-weight: bold;
    font-size: 18px;
}

.signal-values {
    display: flex;
    gap: 2px;
    font-family: 'Courier New', monospace;
}

.signal-bit {
    width: 25px;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid var(--border-primary);
    border-radius: 2px;
    font-weight: bold;
    font-size: 12px;
    color: var(--text-primary);
}

.signal-bit.one {
    background: #10b981;
    color: white;
}

.signal-bit.zero {
    background: var(--bg-tertiary);
    color: var(--text-secondary);
}

.signal-bit.padded {
    opacity: 0.4;
    border-style: dashed;
}

.fft-visualization, .ifft-visualization {
    margin-bottom: 1rem;
}

.fft-grid {
    display: grid;
    gap: 1rem;
    margin-bottom: 1rem;
}

.fft-row {
    background: var(--bg-secondary);
    padding: 1rem;
    border-radius: 6px;
    border: 1px solid var(--border-primary);
}

.fft-label {
    font-weight: bold;
    margin-bottom: 0.5rem;
    color: var(--text-primary);
}

.complex-values {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
}

.complex-number {
    background: var(--bg-primary);
    padding: 4px 8px;
    border-radius: 4px;
    border: 1px solid var(--border-primary);
    font-family: 'Courier New', monospace;
    font-size: 12px;
    color: var(--text-primary);
}

.multiplication-grid {
    margin-bottom: 1rem;
}

.multiply-row {
    display: grid;
    grid-template-columns: 1fr auto 1fr auto 1fr;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1rem;
    padding: 1rem;
    background: var(--bg-secondary);
    border-radius: 6px;
}

.multiply-operator {
    font-size: 20px;
    font-weight: bold;
    color: var(--text-secondary);
    text-align: center;
}

.results-display {
    margin-bottom: 1rem;
}

.results-grid {
    display: grid;
    gap: 1rem;
}

.character-result {
    background: var(--bg-secondary);
    padding: 1rem;
    border-radius: 6px;
    border: 1px solid var(--border-primary);
}

.character-label {
    font-weight: bold;
    margin-bottom: 0.5rem;
    color: var(--text-primary);
}

.correlation-values {
    display: flex;
    gap: 4px;
    margin-bottom: 0.5rem;
}

.correlation-value {
    width: 35px;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 3px;
    font-family: 'Courier New', monospace;
    font-size: 11px;
    font-weight: bold;
    color: var(--text-primary);
}

.final-scores {
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-tertiary);
    border: 2px solid var(--accent-primary);
    border-radius: 6px;
}

.final-scores h4 {
    margin: 0 0 0.5rem 0;
    color: var(--accent-primary);
}

.score-values {
    display: flex;
    gap: 4px;
}

.score-value {
    width: 35px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 2px solid var(--accent-primary);
    border-radius: 4px;
    font-family: 'Courier New', monospace;
    font-weight: bold;
    color: var(--accent-primary);
}

.score-value.perfect-match {
    background: #10b981;
    border-color: #10b981;
    color: white;
    animation: pulse 1s infinite;
}

@keyframes pulse {
    0%, 100% { transform: scale(1); }
    50% { transform: scale(1.1); }
}

.step-explanation {
    background: var(--bg-tertiary);
    padding: 1rem;
    border-radius: 6px;
    color: var(--text-primary);
    font-style: italic;
    border-left: 4px solid var(--accent-primary);
}

.demo-progress {
    background: var(--bg-primary);
    padding: 1rem;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    border: 1px solid var(--border-primary);
}

.progress-bar {
    width: 100%;
    height: 8px;
    background: var(--bg-tertiary);
    border-radius: 4px;
    overflow: hidden;
    margin-bottom: 0.5rem;
}

.progress-fill {
    height: 100%;
    background: linear-gradient(90deg, var(--accent-primary), #10b981);
    width: 0%;
    transition: width 0.3s ease;
}

.progress-text {
    text-align: center;
    font-weight: 600;
    color: var(--text-primary);
}

/* Mobile responsiveness for FFT demo */
@media (max-width: 768px) {
    .fft-demo-container {
        padding: 1rem;
        margin: 1.5rem 0;
    }
    
    .demo-controls {
        padding: 1rem;
    }
    
    .demo-buttons {
        flex-direction: column;
        gap: 0.75rem;
    }
    
    .demo-buttons button {
        width: 100%;
        justify-content: center;
    }
    
    .input-group input {
        font-size: 16px; /* Prevents zoom on iOS */
    }
    
    .multiply-row {
        grid-template-columns: 1fr;
        text-align: center;
        gap: 0.75rem;
    }
    
    .complex-values, .correlation-values, .score-values {
        justify-content: center;
        flex-wrap: wrap;
        gap: 3px;
    }
    
    .complex-number, .correlation-value, .score-value {
        font-size: 10px;
        padding: 2px 4px;
    }
    
    .sequence-char, .signal-bit {
        width: 20px;
        height: 20px;
        font-size: 10px;
    }
    
    .step-content {
        padding: 1rem;
    }
    
    .signals-grid {
        gap: 0.75rem;
    }
    
    .signal-row {
        flex-wrap: wrap;
        gap: 0.5rem;
    }
    
    .character-result {
        padding: 0.75rem;
    }
}

@media (max-width: 480px) {
    .fft-demo-container {
        padding: 0.75rem;
    }
    
    .demo-controls {
        padding: 0.75rem;
    }
    
    .complex-number, .correlation-value {
        width: 30px;
        height: 20px;
        font-size: 9px;
        padding: 1px 2px;
    }
    
    .score-value {
        width: 30px;
        height: 25px;
        font-size: 10px;
    }
    
    .sequence-char, .signal-bit {
        width: 18px;
        height: 18px;
        font-size: 9px;
    }
    
    .final-scores h4 {
        font-size: 14px;
    }
}
</style>

<script>
class FFTPatternMatchingDemo {
    constructor() {
        this.currentStep = 0;
        this.isRunning = false;
        this.stepMode = false;
        this.textSignals = {};
        this.patternSignals = {};
        this.textFFT = {};
        this.patternFFT = {};
        this.multipliedFFT = {};
        this.correlations = {};
        this.finalScores = [];
        
        this.steps = [
            'input', 'signals', 'text-fft', 'pattern-fft', 
            'multiply', 'ifft', 'results'
        ];
    }

    init() {
        document.getElementById('fft-run-demo').addEventListener('click', () => this.runCompleteDemo());
        document.getElementById('fft-step-demo').addEventListener('click', () => this.stepThroughDemo());
        document.getElementById('fft-reset-demo').addEventListener('click', () => this.resetDemo());
        
        document.getElementById('fft-text').addEventListener('input', () => this.resetDemo());
        document.getElementById('fft-pattern').addEventListener('input', () => this.resetDemo());
        
        this.resetDemo();
    }

    async runCompleteDemo() {
        if (this.isRunning) return;
        this.isRunning = true;
        this.stepMode = false;
        
        for (let i = 0; i < this.steps.length; i++) {
            await this.executeStep(i);
            await this.sleep(1500);
        }
        
        this.isRunning = false;
    }

    stepThroughDemo() {
        if (this.isRunning) return;
        this.stepMode = true;
        
        if (this.currentStep < this.steps.length) {
            this.executeStep(this.currentStep);
            this.currentStep++;
        }
    }

    resetDemo() {
        this.isRunning = false;
        this.stepMode = false;
        this.currentStep = 0;
        
        // Hide all steps
        this.steps.forEach(step => {
            document.getElementById(`step-${step}`).classList.remove('active');
        });
        
        this.updateProgress(0);
        this.processInput();
    }

    async executeStep(stepIndex) {
        // Activate current step
        this.steps.forEach((step, index) => {
            const element = document.getElementById(`step-${step}`);
            if (index === stepIndex) {
                element.classList.add('active');
                element.scrollIntoView({ behavior: 'smooth', block: 'center' });
            } else if (index < stepIndex) {
                element.classList.remove('active');
                element.style.opacity = '0.7';
            }
        });

        this.updateProgress((stepIndex + 1) / this.steps.length * 100);

        switch (stepIndex) {
            case 0: await this.stepInput(); break;
            case 1: await this.stepSignals(); break;
            case 2: await this.stepTextFFT(); break;
            case 3: await this.stepPatternFFT(); break;
            case 4: await this.stepMultiply(); break;
            case 5: await this.stepIFFT(); break;
            case 6: await this.stepResults(); break;
        }
    }

    processInput() {
        const text = document.getElementById('fft-text').value.toUpperCase().replace(/[^ATGC]/g, '').substring(0, 12);
        const pattern = document.getElementById('fft-pattern').value.toUpperCase().replace(/[^ATGC]/g, '').substring(0, 6);
        
        // Pad to power of 2 for FFT, ensure we can see the full padding
        const n = Math.max(text.length + pattern.length - 1, 8);
        const paddedLength = Math.pow(2, Math.ceil(Math.log2(n)));
        
        this.text = text;
        this.pattern = pattern;
        this.paddedLength = paddedLength;
        
        // Update input fields
        document.getElementById('fft-text').value = text;
        document.getElementById('fft-pattern').value = pattern;
    }

    async stepInput() {
        this.processInput();
        
        const textDisplay = document.getElementById('text-display');
        const patternDisplay = document.getElementById('pattern-display');
        
        textDisplay.innerHTML = this.text.split('').map(char => 
            `<div class="sequence-char ${char}">${char}</div>`
        ).join('');
        
        patternDisplay.innerHTML = this.pattern.split('').map(char => 
            `<div class="sequence-char ${char}">${char}</div>`
        ).join('');
        
        document.getElementById('input-explanation').innerHTML = 
            `We have text "${this.text}" (${this.text.length} characters) and pattern "${this.pattern}" (${this.pattern.length} characters). We'll pad to ${this.paddedLength} for FFT efficiency.`;
    }

    async stepSignals() {
        const bases = ['A', 'T', 'G', 'C'];
        const colors = ['#dc2626', '#2563eb', '#16a34a', '#d97706'];
        
        // Generate signals
        bases.forEach(base => {
            this.textSignals[base] = this.createPaddedSignal(this.text, base);
            this.patternSignals[base] = this.createPaddedSignal(this.pattern, base);
        });
        
        const grid = document.getElementById('signals-grid');
        grid.innerHTML = '';
        
        bases.forEach((base, index) => {
            // Text signal
            const textRow = document.createElement('div');
            textRow.className = 'signal-row';
            textRow.innerHTML = `
                <div class="signal-label" style="color: ${colors[index]}">T${base}:</div>
                <div class="signal-values">
                    ${this.textSignals[base].map((bit, i) => 
                        `<div class="signal-bit ${bit ? 'one' : 'zero'} ${i >= this.text.length ? 'padded' : ''}">${bit}</div>`
                    ).join('')}
                </div>
            `;
            grid.appendChild(textRow);
            
            // Pattern signal (normal order)
            const patternSignal = new Array(this.paddedLength).fill(0);
            for (let i = 0; i < this.pattern.length; i++) {
                if (this.pattern[i] === base) {
                    patternSignal[i] = 1;
                }
            }
            const patternRow = document.createElement('div');
            patternRow.className = 'signal-row';
            patternRow.innerHTML = `
                <div class="signal-label" style="color: ${colors[index]}">P${base}:</div>
                <div class="signal-values">
                    ${patternSignal.map((bit, i) => 
                        `<div class="signal-bit ${bit ? 'one' : 'zero'} ${i >= this.pattern.length ? 'padded' : ''}">${bit}</div>`
                    ).join('')}
                </div>
            `;
            grid.appendChild(patternRow);
        });
        
        document.getElementById('signals-explanation').innerHTML = 
            `Each base (A,T,G,C) gets its own binary signal. Pattern signals are reversed for cross-correlation. Signals are padded to ${this.paddedLength} for FFT.`;
    }

    async stepTextFFT() {
        const bases = ['A', 'T', 'G', 'C'];
        
        // Calculate FFT for each text signal
        bases.forEach(base => {
            this.textFFT[base] = this.fft(this.textSignals[base]);
        });
        
        const viz = document.getElementById('text-fft-viz');
        viz.innerHTML = '<div class="fft-grid"></div>';
        const grid = viz.querySelector('.fft-grid');
        
        bases.forEach(base => {
            const row = document.createElement('div');
            row.className = 'fft-row';
            row.innerHTML = `
                <div class="fft-label">FFT(T${base})</div>
                <div class="complex-values">
                    ${this.textFFT[base].map(complex => 
                        `<div class="complex-number">${this.formatComplex(complex)}</div>`
                    ).join('')}
                </div>
            `;
            grid.appendChild(row);
        });
        
        document.getElementById('text-fft-explanation').innerHTML = 
            `Text signals transformed to frequency domain. Each complex number represents a frequency component with magnitude and phase.`;
    }

    async stepPatternFFT() {
        const bases = ['A', 'T', 'G', 'C'];
        
        // Calculate FFT for each pattern signal 
        bases.forEach(base => {
            // Create pattern signal (no reversal needed)
            const patternSignal = new Array(this.paddedLength).fill(0);
            for (let i = 0; i < this.pattern.length; i++) {
                if (this.pattern[i] === base) {
                    patternSignal[i] = 1;
                }
            }
            this.patternFFT[base] = this.fft(patternSignal);
        });
        
        const viz = document.getElementById('pattern-fft-viz');
        viz.innerHTML = '<div class="fft-grid"></div>';
        const grid = viz.querySelector('.fft-grid');
        
        bases.forEach(base => {
            const row = document.createElement('div');
            row.className = 'fft-row';
            row.innerHTML = `
                <div class="fft-label">FFT(P${base}_rev)</div>
                <div class="complex-values">
                    ${this.patternFFT[base].map(complex => 
                        `<div class="complex-number">${this.formatComplex(complex)}</div>`
                    ).join('')}
                </div>
            `;
            grid.appendChild(row);
        });
        
        document.getElementById('pattern-fft-explanation').innerHTML = 
            `Reversed pattern signals transformed to frequency domain. Reversing gives us cross-correlation instead of convolution.`;
    }

    async stepMultiply() {
        const bases = ['A', 'T', 'G', 'C'];
        
        // Multiply FFTs element-wise (text FFT * conjugate of pattern FFT for correlation)
        bases.forEach(base => {
            this.multipliedFFT[base] = [];
            for (let i = 0; i < this.textFFT[base].length; i++) {
                const textComp = this.textFFT[base][i];
                const patternComp = this.patternFFT[base][i];
                // Take conjugate of pattern for correlation
                const patternConj = { real: patternComp.real, imag: -patternComp.imag };
                this.multipliedFFT[base][i] = this.multiplyComplex(textComp, patternConj);
            }
        });
        
        const grid = document.getElementById('multiply-grid');
        grid.innerHTML = '';
        
        bases.forEach(base => {
            const row = document.createElement('div');
            row.className = 'multiply-row';
            row.innerHTML = `
                <div>
                    <div style="font-weight: bold; margin-bottom: 0.5rem;">FFT(T${base})</div>
                    <div class="complex-values">
                        ${this.textFFT[base].map(c => 
                            `<div class="complex-number">${this.formatComplex(c)}</div>`
                        ).join('')}
                    </div>
                </div>
                <div class="multiply-operator">×</div>
                <div>
                    <div style="font-weight: bold; margin-bottom: 0.5rem;">FFT(P${base})</div>
                    <div class="complex-values">
                        ${this.patternFFT[base].map(c => 
                            `<div class="complex-number">${this.formatComplex(c)}</div>`
                        ).join('')}
                    </div>
                </div>
                <div class="multiply-operator">=</div>
                <div>
                    <div style="font-weight: bold; margin-bottom: 0.5rem;">Result</div>
                    <div class="complex-values">
                        ${this.multipliedFFT[base].map(c => 
                            `<div class="complex-number">${this.formatComplex(c)}</div>`
                        ).join('')}
                    </div>
                </div>
            `;
            grid.appendChild(row);
        });
        
        document.getElementById('multiply-explanation').innerHTML = 
            `Element-wise multiplication in frequency domain. This is equivalent to convolution in time domain - the magic of the convolution theorem!`;
    }

    async stepIFFT() {
        const bases = ['A', 'T', 'G', 'C'];
        
        // Calculate IFFT to get correlations
        bases.forEach(base => {
            const ifftResult = this.ifft(this.multipliedFFT[base]);
            const correlationResults = ifftResult.map(c => Math.round(c.real));
            
            // Extract valid correlation positions (first part of result)
            const validPositions = this.text.length - this.pattern.length + 1;
            this.correlations[base] = correlationResults.slice(0, validPositions);
        });
        
        const viz = document.getElementById('ifft-viz');
        viz.innerHTML = '<div class="fft-grid"></div>';
        const grid = viz.querySelector('.fft-grid');
        
        bases.forEach(base => {
            const row = document.createElement('div');
            row.className = 'fft-row';
            row.innerHTML = `
                <div class="fft-label">IFFT → Correlation for ${base}</div>
                <div class="correlation-values">
                    ${this.correlations[base].slice(0, this.text.length - this.pattern.length + 1).map(val => 
                        `<div class="correlation-value">${val}</div>`
                    ).join('')}
                </div>
            `;
            grid.appendChild(row);
        });
        
        document.getElementById('ifft-explanation').innerHTML = 
            `Inverse FFT transforms back to time domain. Each value shows how many characters of that type match at each position.`;
    }

    async stepResults() {
        const bases = ['A', 'T', 'G', 'C'];
        
        // Calculate final scores by summing correlations
        this.finalScores = [];
        const maxValidPosition = this.text.length - this.pattern.length + 1;
        for (let i = 0; i < maxValidPosition; i++) {
            let score = 0;
            bases.forEach(base => {
                score += this.correlations[base][i] || 0;
            });
            this.finalScores.push(score);
        }
        
        const display = document.getElementById('results-display');
        display.innerHTML = `
            <div class="results-grid">
                ${bases.map(base => `
                    <div class="character-result">
                        <div class="character-label">Character ${base} Matches:</div>
                        <div class="correlation-values">
                            ${this.correlations[base].slice(0, this.text.length - this.pattern.length + 1).map(val => 
                                `<div class="correlation-value">${val}</div>`
                            ).join('')}
                        </div>
                    </div>
                `).join('')}
            </div>
            <div class="final-scores">
                <h4>Final Match Scores (Sum of all characters):</h4>
                <div class="score-values">
                    ${this.finalScores.map((score, i) => 
                        `<div class="score-value ${score === this.pattern.length ? 'perfect-match' : ''}">${score}</div>`
                    ).join('')}
                </div>
                <div style="margin-top: 1rem; font-size: 14px;">
                    Perfect matches (score = ${this.pattern.length}): ${this.finalScores.map((score, i) => 
                        score === this.pattern.length ? `position ${i}` : null
                    ).filter(x => x).join(', ') || 'none'}
                </div>
                <div style="margin-top: 0.5rem; font-size: 12px; color: #6b7280;">
                    Position indices start from 0. Each position shows where the pattern would align in the text.
                </div>
            </div>
        `;
        
        document.getElementById('results-explanation').innerHTML = 
            `Final pattern matching complete! Positions with score = ${this.pattern.length} are perfect matches. Lower scores indicate partial matches.`;
    }

    // Helper functions
    createPaddedSignal(sequence, char) {
        const signal = sequence.split('').map(c => c === char ? 1 : 0);
        while (signal.length < this.paddedLength) {
            signal.push(0);
        }
        return signal;
    }

    fft(input) {
        // Convert input to complex numbers if needed
        const signal = input.map(x => 
            typeof x === 'number' ? { real: x, imag: 0 } : x
        );
        
        const n = signal.length;
        if (n <= 1) return signal;
        
        const even = [];
        const odd = [];
        for (let i = 0; i < n; i += 2) {
            even.push(signal[i]);
            if (i + 1 < n) odd.push(signal[i + 1]);
        }
        
        const evenFFT = this.fft(even);
        const oddFFT = this.fft(odd);
        
        const result = new Array(n);
        for (let k = 0; k < n / 2; k++) {
            const angle = -2 * Math.PI * k / n;
            const wk = { real: Math.cos(angle), imag: Math.sin(angle) };
            const oddTerm = this.multiplyComplex(wk, oddFFT[k] || { real: 0, imag: 0 });
            
            result[k] = this.addComplex(evenFFT[k], oddTerm);
            result[k + n / 2] = this.subtractComplex(evenFFT[k], oddTerm);
        }
        
        return result;
    }

    ifft(complexArray) {
        // Conjugate input, do FFT, conjugate output, scale
        const conjugated = complexArray.map(c => ({ real: c.real, imag: -c.imag }));
        const fftResult = this.fft(conjugated);
        return fftResult.map(c => ({ 
            real: c.real / complexArray.length, 
            imag: -c.imag / complexArray.length 
        }));
    }

    multiplyComplex(a, b) {
        return {
            real: a.real * b.real - a.imag * b.imag,
            imag: a.real * b.imag + a.imag * b.real
        };
    }

    addComplex(a, b) {
        return { real: a.real + b.real, imag: a.imag + b.imag };
    }

    subtractComplex(a, b) {
        return { real: a.real - b.real, imag: a.imag - b.imag };
    }

    formatComplex(complex) {
        const real = complex.real.toFixed(1);
        const imag = complex.imag.toFixed(1);
        if (Math.abs(complex.imag) < 0.1) return real;
        if (Math.abs(complex.real) < 0.1) return `${imag}i`;
        return `${real}${complex.imag >= 0 ? '+' : ''}${imag}i`;
    }

    updateProgress(percent) {
        document.getElementById('progress-fill').style.width = `${percent}%`;
        const stepNames = ['Ready', 'Input', 'Signals', 'Text FFT', 'Pattern FFT', 'Multiply', 'IFFT', 'Results'];
        document.getElementById('progress-text').textContent = 
            percent === 0 ? 'Ready to start' : 
            percent === 100 ? 'Demo complete!' : 
            `Step ${Math.floor(percent / 100 * 7) + 1}: ${stepNames[Math.floor(percent / 100 * 7) + 1]}`;
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Initialize demo when page loads
document.addEventListener('DOMContentLoaded', () => {
    const demo = new FFTPatternMatchingDemo();
    demo.init();
});
</script>

---

And there you have it! From the humble beginnings of a brute force search taking 11.6 days, we've journeyed through the mathematical wonderland of signal processing, Fourier transforms, and convolution theory to arrive at an algorithm that can solve the same problem in just 5 minutes.

This isn't just a story about optimization—it's a perfect example of how **mathematical abstractions** can lead to breakthrough solutions. By recognizing that pattern matching(good old sliding window approach) is really just convolution in disguise, and that convolution becomes multiplication in the frequency domain, we unlocked the power of FFT to transform an intractable problem into a manageable one.

Whether you're searching for DNA sequences in massive genomes, finding patterns in time series data, or implementing any kind of template matching, the principles we've explored here will serve you well.


## Conclusion

We went from brute force (11.6 days) to FFT magic (5 minutes) - but this wasn't just about speed.

We explored **trees** (tries), **graphs** (Aho-Corasick automata), and **mathematics** (signal processing). Each taught us something different about algorithmic thinking. And let's take a moment to appreciate that we did the string pattern matching in frequency domain. 

What I want you to take away from this is how very similar looking problems can have extremely different optimal solutions. And we might need to take completely different approaches to solve each of them, maybe even go out of our way a little bit.


Thanks for reading! I hope you've learned something new and caught a glimpse of how trees, graphs, and math can solve problems in unexpected ways. The next time you hit a wall, don't just code harder—think differently!

