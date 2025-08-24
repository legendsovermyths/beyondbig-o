---
layout: post
title: "Autocomplete on Steroids: Building a Fuzzy Search Engine That Actually Scales"
date: 2024-03-15
author: "Anirudh Singh"
excerpt: "How do you search for 5,000 keywords (with typos!) across 100,000 documents in under 100ms? Spoiler: Not with nested loops. A journey through tries, automata theory, and signal processing that achieves an 84,000x speedup."
tags: ["algorithms", "automata-theory", "fft", "text-processing", "aho-corasick", "interactive"]
reading_time: 25
---
 
Picture this: you're at a major tech company, and your team just got handed a nightmare of a project. 
 
"We need autocomplete for our documentation search," your PM says. "Should be simple, right?"
 
Sure, if by "simple" they mean:
- Search through **100,000 technical documents** (Stack Overflow posts, API docs, tutorials)
- Find any of **5,000 keywords** (every major framework, library, and technical term)
- Handle typos because developers can't type (find "JavaScriot" when they meant "JavaScript")
- Return results in under 100ms because nobody likes waiting
 
Oh, and each document averages **1,000 characters**. That's 100 million characters of text to search through.
 
I lean back in my chair, doing some quick math. "So we need to find 5,000 needles in a 100-million-character haystack... with typos?"
 
"Exactly! When can you have it done?"
 
*nervous laughter*
 
## The Scale of the Problem
 
Let me break down what we're really dealing with here:
 
- **5,000 keywords** to search for (think "React", "Python", "PostgreSQL", etc.)
- **100,000 documents** to search through
- **1,000 characters** per document on average
- **Fuzzy matching** for single-character typos
 
So "Photoshop" needs to match not just "Photoshop", but also "Photoshkp", "Phatoshop", "Photoshxp"... basically any version where exactly one character is different.
 
For a 9-character word like "Photoshop", that's about 93×9 = 837 variations (each position can be any of 93 other printable ASCII characters).
 
Let's see... 5,000 keywords × ~837 typo variants each = about 4.2 million strings to search for.
 
*Cool cool cool.*
 
## But What Are We Actually Building Here?
 
Before we dive into solutions, let me tell you the endgame. We're not just looking for a "yes/no" answer about whether patterns exist. We're building something much more powerful.
 
**We're going to create a score vector** - a magical array that tells us, for every position in our text, exactly how many characters of our pattern match at that position. 
 
Think about it:
- Score = pattern length? That's an exact match!
- Score = pattern length - 1? That's a fuzzy match with one typo!
- Score = pattern length - 2? Two typos!
 
This score vector is the key to everything. Once we have it, fuzzy matching becomes trivial - just look for positions where the score is "close enough" to the pattern length.
 
The journey we're about to take - through tries, automata, and finally signal processing - is all about computing this score vector efficiently. And by the end, we'll do it in O(N log N) time using FFT, which seems like straight-up wizardry.
 
## The "Obvious" Solution
 
Alright, so first things first — let me show you what I tried initially, because I'm sure you're thinking the same thing.
 
Just loop through every document, then loop through every keyword, and check if it's there. Classic nested loops. Your CS professor would be proud.
 
```cpp
vector<pair<string, string>> search_documents_naive(
    const vector<string>& documents, 
    const vector<string>& keywords) {
    
    vector<pair<string, string>> results;
    
    for (const auto& doc : documents) {        // 100,000 documents
        for (const auto& keyword : keywords) { // 5,000 keywords
            if (doc.find(keyword) != string::npos) { // String search
                results.emplace_back(doc, keyword);
            }
        }
    }
    return results;
}
```
 
So that's **O(D × K × N)** complexity. Plugging in our numbers: 100,000 × 5,000 × 1,000 = 500 billion operations.
 
My computer would take about 8 minutes to finish. Not terrible.
 
Oh wait, I almost forgot about the fuzzy matching part. 
 
*nervous laughter* 
 
So now every keyword needs to generate all its possible typo variations first.
 
```cpp
vector<string> generate_typos(const string& word) {
    vector<string> variations;
    variations.push_back(word);  // Original
    
    // All printable ASCII characters (space through ~)
    const string charset = " !\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\n]^_`abcdefghijklmnopqrstuvwxyz{|}~";
    
    // Single character substitutions: "photoshop" -> "phatoshop", "photosh0p", "photosh@p", etc.
    for (size_t i = 0; i < word.length(); ++i) {
        for (char c : charset) {
            if (c != word[i]) {
                string variant = word;
                variant[i] = c;
                variations.push_back(variant);
            }
        }
    }
    
    return variations;
}
```
 
Now we're looking at **O(D × K × V × N)** where V is about 837 variations per keyword.
 
100,000 × 5,000 × 837 × 1,000 = 420 trillion operations.
 
Yeah, that's not happening. At 1 billion operations per second, we're looking at about 5 days of computation. For a single search.
 
Here, let me show you just how spectacularly this approach fails:
 
<div class="naive-demo">
    <div id="naive-visualization"></div>
    <div class="demo-controls">
        <label>Documents: <span id="doc-count">100000</span></label>
        <input type="range" id="doc-slider" min="1000" max="100000" value="100000" step="1000">
        
        <label>Keywords: <span id="keyword-count">5000</span></label>
        <input type="range" id="keyword-slider" min="100" max="5000" value="5000" step="100">
        
        <label>Fuzzy matching: <input type="checkbox" id="fuzzy-toggle" checked></label>
    </div>
    <div id="computation-stats"></div>
</div>
 
<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js"></script>
<script>
let naiveSketch = function(p) {
    let docCount = 100000;
    let keywordCount = 5000;
    let fuzzyEnabled = true;
    let animationFrame = 0;
    
    p.setup = function() {
        let canvas = p.createCanvas(700, 400);
        canvas.parent('naive-visualization');
        p.frameRate(30);
    };
    
    p.draw = function() {
        let bgColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--bg-primary').trim();
        let accentColor = getComputedStyle(document.documentElement)
            .getPropertyValue('--accent-primary').trim();
        
        p.background(bgColor);
        
        // Fixed grid size - always show same visual density
        let maxDisplayDocs = 40;
        let maxDisplayKeywords = 20;
        let cellWidth = 12;
        let cellHeight = 12;
        let spacing = 2;
        
        // Calculate how many we can actually display
        let displayDocs = Math.min(docCount, maxDisplayDocs);
        let displayKeywords = Math.min(keywordCount, maxDisplayKeywords);
        
        // Animate through the nested loops
        let currentDoc = Math.floor(animationFrame / 30) % displayDocs;
        let currentKeyword = Math.floor(animationFrame / 3) % displayKeywords;
        
        // Draw the grid
        for (let d = 0; d < displayDocs; d++) {
            for (let k = 0; k < displayKeywords; k++) {
                let x = 50 + d * (cellWidth + spacing);
                let y = 50 + k * (cellHeight + spacing);
                
                // Color based on current position
                if (d === currentDoc && k === currentKeyword) {
                    p.fill('#ef4444'); // Current operation - red
                } else if (d < currentDoc || (d === currentDoc && k < currentKeyword)) {
                    p.fill('#22c55e'); // Completed - green
                } else {
                    p.fill(accentColor); // Not yet processed
                }
                
                p.rect(x, y, cellWidth, cellHeight);
            }
        }
        
        // Show scale indicator
        p.fill('#94a3b8');
        p.textAlign(p.LEFT);
        p.textSize(11);
        p.text(`Showing ${displayDocs} × ${displayKeywords} operations`, 50, 35);
        
        // Show overflow indicator if we have way more
        if (docCount > maxDisplayDocs || keywordCount > maxDisplayKeywords) {
            let totalHidden = docCount * keywordCount - displayDocs * displayKeywords;
            p.textAlign(p.CENTER);
            p.textSize(14);
            p.fill('#ef4444');
            p.text(`... + ${totalHidden.toLocaleString()} more operations not shown`, 
                   p.width/2, p.height - 30);
            
            // Add dramatic effect for really large numbers
            if (totalHidden > 1000000) {
                p.textSize(10);
                p.fill('#94a3b8');
                p.text('(This is why your CPU would melt)', p.width/2, p.height - 15);
            }
        }
        
        animationFrame++;
    };
    
    // Global functions for controls
    window.updateNaiveViz = function() {
        docCount = parseInt(document.getElementById('doc-slider').value);
        keywordCount = parseInt(document.getElementById('keyword-slider').value);
        fuzzyEnabled = document.getElementById('fuzzy-toggle').checked;
        
        document.getElementById('doc-count').textContent = docCount.toLocaleString();
        document.getElementById('keyword-count').textContent = keywordCount.toLocaleString();
        
        // Calculate total operations
        let variations = fuzzyEnabled ? 837 : 1;
        let totalOps = docCount * keywordCount * variations * 1000; // 1000 chars avg
        
        let timeEstimate = totalOps / 1000000; // Assume 1M ops/second
        let timeUnit = 'seconds';
        
        if (timeEstimate > 60) {
            timeEstimate /= 60;
            timeUnit = 'minutes';
        }
        if (timeEstimate > 60) {
            timeEstimate /= 60;
            timeUnit = 'hours';
        }
        if (timeEstimate > 24) {
            timeEstimate /= 24;
            timeUnit = 'days';
        }
        if (timeEstimate > 365) {
            timeEstimate /= 365;
            timeUnit = 'years';
        }
        
        document.getElementById('computation-stats').innerHTML = `
            <strong>Total Operations:</strong> ${totalOps.toLocaleString()}<br>
            <strong>Estimated Time:</strong> ${timeEstimate.toFixed(1)} ${timeUnit}
            ${timeEstimate > 1000 ? ' <em>(basically forever)</em>' : ''}
        `;
    };
};
 
new p5(naiveSketch);
 
// Event listeners
document.getElementById('doc-slider').addEventListener('input', window.updateNaiveViz);
document.getElementById('keyword-slider').addEventListener('input', window.updateNaiveViz);
document.getElementById('fuzzy-toggle').addEventListener('change', window.updateNaiveViz);
 
// Initial update
window.updateNaiveViz();
</script>
 
<style>
.naive-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-color);
}
 
.demo-controls {
    display: flex;
    gap: 2rem;
    align-items: center;
    justify-content: center;
    margin: 1rem 0;
    flex-wrap: wrap;
}
 
.demo-controls label {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-weight: 500;
}
 
.demo-controls input[type="range"] {
    width: 150px;
}
 
#computation-stats {
    text-align: center;
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
}
</style>
 
Try adjusting the sliders in that demo. With our target of 100,000 documents and 5,000 keywords, fuzzy matching enabled, we're at 420 trillion operations. Each red square represents your CPU having a small existential crisis.
 
Even if you scale it down to just 10,000 documents and 500 keywords, you're still looking at billions of operations. This approach is about as effective as trying to empty the ocean with a teaspoon.
 
---
 
## Enter the Trie (And Why It's Actually Genius)
 
So I'm sitting there, watching my CPU slowly die, when I remembered something from my competitive programming days. There's this data structure called a trie — basically a tree where each path from root to leaf spells out a word.
 
And then it hit me: "Wait, what if instead of checking each keyword separately, I build one massive tree containing all my keywords and just walk through it once?"
 
This is where things start getting interesting. Here's an interactive trie you can mess around with:
 
<div class="trie-demo">
    <div class="trie-controls">
        <div class="input-section">
            <input type="text" id="trie-word-input" placeholder="Type a word to add" maxlength="15">
            <button id="add-word-btn">Add Word</button>
            <button id="clear-trie-btn">Clear All</button>
        </div>
        <div class="search-section">
            <input type="text" id="search-input" placeholder="Search for a word" maxlength="15">
            <button id="search-btn">Search</button>
            <span id="search-result"></span>
        </div>
        <div class="preset-section">
            <button class="preset-btn" data-words="cat,car,card">Add: cat, car, card</button>
            <button class="preset-btn" data-words="photoshop,photo,phone">Add: photoshop, photo, phone</button>
            <button class="preset-btn" data-words="code,coder,coding">Add: code, coder, coding</button>
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
        // Create normal resolution canvas
        let canvas = p.createCanvas(800, 500);
        canvas.parent('trie-visualization');
        
        // Enable smooth rendering and high DPI
        p.pixelDensity(Math.min(window.devicePixelRatio || 1, 2));
        p.smooth();
        
        // Force high-quality rendering
        canvas.canvas.style.imageRendering = 'auto';
        canvas.canvas.style.imageRendering = 'high-quality';
        
        // Add some initial words
        console.log('Adding initial words to trie...');
        trie.insert('cat');
        trie.insert('car');
        trie.insert('card');
        console.log('Trie word count:', trie.wordCount);
        updateTrieStats();
        
        // Update colors after DOM is ready
        setTimeout(updateColors, 100);
    };
    
    function updateColors() {
        try {
            colors.bg = getComputedStyle(document.documentElement)
                .getPropertyValue('--bg-primary').trim() || '#ffffff';
            const computedAccent = getComputedStyle(document.documentElement)
                .getPropertyValue('--text-primary').trim();
            colors.accent = computedAccent || '#111111';
            const computedText = getComputedStyle(document.documentElement)
                .getPropertyValue('--text-primary').trim();
            colors.text = computedText || '#111111';
        } catch (e) {
            console.log('Using fallback colors');
        }
    }
    
    p.draw = function() {
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
        
        // Scale based on content
        const availableHeight = p.height - 160; // Leave space for legend
        const availableWidth = p.width - 100;
        const levelHeight = Math.min(80, availableHeight / (maxLevel + 1));
        const minNodeSpacing = 60;
        
        for (let [level, levelNodes] of levels.entries()) {
            const y = 50 + level * levelHeight;
            
            if (levelNodes.length === 1) {
                levelNodes[0].targetX = p.width / 2;
                levelNodes[0].targetY = y;
            } else {
                // Calculate spacing to fit all nodes with proper centering
                const neededWidth = (levelNodes.length - 1) * minNodeSpacing;
                const actualWidth = Math.min(neededWidth, availableWidth);
                const spacing = levelNodes.length > 1 ? actualWidth / (levelNodes.length - 1) : 0;
                const startX = p.width / 2 - actualWidth / 2;
                
                levelNodes.forEach((node, index) => {
                    node.targetX = startX + index * spacing;
                    node.targetY = y;
                });
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
                // Subtle green for animated search, otherwise monochrome
                p.stroke(toNode.searchHighlight ? '#16a34a' : '#111111');
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
                p.fill('#ffffff');
                p.stroke('#111111');
            } else if (node === trie.root) {
                p.fill('#ffffff');
                p.stroke('#111111');
            } else {
                p.fill('#ffffff');
                p.stroke('#111111');
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
                p.fill('#111111');
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
        
        p.fill('#ffffff');
        p.stroke('#e5e7eb');
        p.strokeWeight(1);
        p.rect(legendX - 10, legendY - 10, 190, 100, 8);
        
        p.fill('#374151');
        p.textAlign(p.LEFT);
        p.textSize(12);
        p.noStroke();
        
        // Root node
        p.fill('#ffffff');
        p.circle(legendX + 10, legendY + 5, 15);
        p.stroke('#111111');
        p.noFill();
        p.circle(legendX + 10, legendY + 5, 15);
        p.noStroke();
        p.fill('#374151');
        p.text('Root node', legendX + 30, legendY + 8);
        
        // Regular node
        p.fill('#ffffff');
        p.circle(legendX + 10, legendY + 25, 15);
        p.stroke('#111111');
        p.noFill();
        p.circle(legendX + 10, legendY + 25, 15);
        p.noStroke();
        p.fill('#374151');
        p.text('Character node', legendX + 30, legendY + 28);
        
        // End of word
        p.fill('#ffffff');
        p.circle(legendX + 10, legendY + 45, 15);
        p.stroke('#111111');
        p.noFill();
        p.circle(legendX + 10, legendY + 45, 15);
        p.noStroke();
        p.fill('#111111');
        p.circle(legendX + 16, legendY + 39, 6);
        p.fill('#374151');
        p.text('End of word', legendX + 30, legendY + 48);
        
        // Search highlight
        p.fill('#d1fae5');
        p.circle(legendX + 10, legendY + 65, 15);
        p.stroke('#16a34a');
        p.noFill();
        p.circle(legendX + 10, legendY + 65, 15);
        p.noStroke();
        p.fill('#374151');
        p.text('Search path', legendX + 30, legendY + 68);
    }
    
    // Global functions
    window.addWordToTrie = function(word) {
        if (trie.insert(word)) {
            updateTrieStats();
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
    };
    
    function updateTrieStats() {
        const { nodes } = trie.getAllNodes();
        document.getElementById('trie-stats').innerHTML = `
            <strong>Words stored:</strong> ${trie.wordCount} | 
            <strong>Nodes created:</strong> ${nodes.length - 1} | 
            <strong>Memory saved:</strong> ${calculateMemorySaving()}%
        `;
    }
    
    function calculateMemorySaving() {
        if (trie.wordCount === 0) return 0;
        const { nodes } = trie.getAllNodes();
        const trieNodes = nodes.length - 1; // Exclude root
        const naiveStorage = trie.wordCount * 10; // Assume avg 10 chars per word
        const saving = Math.max(0, Math.round((1 - trieNodes / naiveStorage) * 100));
        return saving;
    }
    
    function updateSearchResult(word, found) {
        const resultEl = document.getElementById('search-result');
        if (found) {
            resultEl.innerHTML = `✅ "${word}" found!`;
            resultEl.style.color = '#22c55e';
        } else {
            resultEl.innerHTML = `❌ "${word}" not found`;
            resultEl.style.color = '#ef4444';
        }
    }
 
    function updateSearchResultFromAnimation() {
        const resultEl = document.getElementById('search-result');
        const word = document.getElementById('search-input').value.trim();
        
        if (trie.searchAnimation.foundWord) {
            resultEl.innerHTML = `✅ "${word}" found!`;
            resultEl.style.color = '#22c55e';
        } else if (trie.searchAnimation.valid) {
            resultEl.innerHTML = `⚠️ "${word}" is a prefix but not a complete word`;
            resultEl.style.color = '#f59e0b';
        } else {
            resultEl.innerHTML = `❌ "${word}" not found`;
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
                alert('Word already exists in trie!');
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
    border: 2px solid var(--border-color);
    border-radius: 6px;
    background: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    min-width: 200px;
}
 
.trie-controls input[type="text"]:focus {
    outline: none;
    border-color: var(--accent-primary);
}
 
.trie-controls button {
    padding: 0.5rem 1rem;
    background: transparent;
    color: #111111;
    border: 1.5px solid #111111;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    font-size: 14px;
    transition: all 0.2s ease;
}
 
.trie-controls button:hover {
    background: #111111;
    color: #ffffff;
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
}
 
#trie-visualization {
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
    margin-bottom: 1rem;
    padding: 1rem;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 500px;
}
 
#trie-visualization canvas {
    display: block;
    image-rendering: auto !important;
    image-rendering: -webkit-optimize-quality !important;
    image-rendering: smooth !important;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    border-radius: 4px;
}
 
#trie-stats {
    text-align: center;
    font-size: 14px;
    color: var(--text-secondary);
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
}
</style>
 
Pretty neat, right? The trie gives us **O(n + m)** complexity where n is the text length and m is the total length of all patterns. That's a massive improvement from our original **O(D × K × N)** nightmare.
 
So now we can walk through a document character by character, following paths in our trie. If a path exists for the current character, great! If not, back to the root we go. 
 
Simple, right?
 
Well... no. Not really.
 
---
 
## Wait, There's a Problem
 
Let me walk you through what actually happens when you use a basic trie. Say we've got "abcde" and "bcde" in our dictionary, and we're searching through the text "abcdf".
 
Here's how it plays out:
1. Start at root
2. See 'a' → follow 'a' path
3. See 'b' → follow 'b' path  
4. See 'c' → follow 'c' path
5. See 'd' → follow 'd' path
6. See 'f' → **no 'f' transition available**
7. Back to root we go
 
But hold on. We just processed "abcd" and then gave up. What about "bcde"? That could totally be a match starting from position 2 in our text! But our trie already moved past those characters.
 
So we're not actually done with those characters. We need to start over from position 2 and check "bcdf", then position 3 and check "cdf", then position 4... you get the idea.
 
This means we're not really getting that beautiful single-pass behavior we wanted. We're still doing multiple passes, just in a slightly smarter way.
 
And it gets worse with overlapping patterns. If we're looking for both "he" and "she" in the text "she sells seashells", our trie will find "she" but completely miss that "he" was sitting right there in positions 2-3.
 
That's when I realized: the trie is great for exact prefix matching, but it's terrible at handling overlaps and making sure we don't miss anything.
 
## Enter Aho-Corasick (Or: How to Never Miss Anything Ever Again)
 
This is where Aho and Corasick showed up in 1975 and basically said, "You know what? Let's fix this properly."
 
Their insight was brilliant: instead of giving up and going back to the root when we can't match a character, what if we had shortcuts that jump us to other parts of the trie where we might still have a chance?
 
These shortcuts are called **failure links**, and they're basically the trie's way of saying "okay, this path didn't work out, but here's the longest suffix of what we've seen so far that could still be the beginning of a match."
 
Let me show you how this works:
 
<div class="aho-corasick-demo">
    <div class="ac-controls">
        <div class="input-section">
            <input type="text" id="ac-text-input" placeholder="Type text to search through" maxlength="50" value="she sells seashells">
            <button id="ac-search-btn">Search</button>
            <button id="ac-step-btn">Step Through</button>
            <button id="ac-reset-btn">Reset</button>
        </div>
        <div class="pattern-section">
            <button class="pattern-btn" data-patterns="he,she,his,hers">Patterns: he, she, his, hers</button>
            <button class="pattern-btn" data-patterns="cat,car,card,care">Patterns: cat, car, card, care</button>
            <button class="pattern-btn" data-patterns="say,she,shr,her">Patterns: say, she, shr, her</button>
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
        
        if (stepMode) {
            return this.searchStep(text);
        }
        
        for (let i = 0; i < text.length; i++) {
            this.processCharacter(text[i].toLowerCase(), i);
        }
        
        return this.matches;
    }
    
    processCharacter(char, position) {
        this.animationState.currentChar = char;
        this.animationState.previousState = this.currentState;
        this.animationState.usedFailureLink = false;
        this.animationState.newMatches = [];
        
        // Clear previous highlights
        this.clearHighlights();
        
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
            .style("background", "#ffffff")
            .style("border-radius", "8px")
            .style("border", "1px solid #e5e7eb");
        
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
            .attr("stroke", "#111111")
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
                if (d.isEndOfWord) return "#ffffff";
                return "#ffffff";
            })
            .attr("stroke", d => {
                if (d.currentMatch) return "#16a34a";
                if (d.highlighted) return "#16a34a";
                return "#111111";
            })
            .attr("stroke-width", 2);
        
        // Add node labels
        this.svg.selectAll("text")
            .data(nodes)
            .enter()
            .append("text")
            .attr("x", d => d.x)
            .attr("y", d => d.y + 5)
            .attr("text-anchor", "middle")
            .attr("fill", "#111111")
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
            .attr("fill", "#111111");
        
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
        const legendData = [
            { color: "#ffffff", stroke: "#111111", text: "State", x: 20, y: this.height - 80 },
            { color: "#ffffff", stroke: "#111111", text: "Accept State", x: 20, y: this.height - 60, marker: true },
            { color: "#d1fae5", stroke: "#16a34a", text: "Current Path", x: 20, y: this.height - 40 },
            { color: "#22c55e", stroke: "#16a34a", text: "Match Found", x: 20, y: this.height - 20 }
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
                    .attr("fill", "#111111");
            }
            
            this.svg.append("text")
                .attr("x", item.x + 25)
                .attr("y", item.y + 4)
                .attr("font-size", "12px")
                .attr("fill", "#374151")
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
            .attr("fill", "#374151")
            .text("Failure Link");
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
        this.ac.processCharacter(char, this.currentPosition);
        
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
    border: 1px solid var(--border-color);
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
    border: 2px solid var(--border-color);
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
}
 
.ac-controls button {
    padding: 0.5rem 1rem;
    background: transparent;
    color: #111111;
    border: 1.5px solid #111111;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    font-size: 14px;
    transition: all 0.2s ease;
}
 
.ac-controls button:hover {
    background: #111111;
    color: #ffffff;
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
    border: 1px solid var(--border-color);
    margin-bottom: 1rem;
    padding: 1rem;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 400px;
}
 
.text-display {
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
}
 
#text-visualization {
    font-family: 'Courier New', monospace;
    font-size: 16px;
    line-height: 1.6;
    word-break: break-all;
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
</style>
 
Try the step-through mode above — it's actually pretty wild to watch. You can see exactly when the failure links kick in and how it manages to find overlapping matches that a basic trie would completely miss.
 
So what just happened here? The Aho-Corasick automaton solved the fundamental problem with our trie approach. Instead of restarting from scratch every time we hit a dead end, those failure links let us maintain all the progress we've made.
 
When we're processing "she" and looking for both "he" and "she", the failure links ensure we're simultaneously tracking both possibilities. No more missed overlaps, no more multiple passes through the same text.
 
The complexity ends up being **O(n + m + z)** where n is text length, m is total pattern length, and z is the number of matches. Compare that to:
- Our original nightmare: O(D × K × V × N) = 420 trillion operations  
- Basic trie: O(n + m) but misses stuff
- Aho-Corasick: O(n + m + z) and catches everything
 
Whether you've got 10 patterns or 5,000 patterns (like in our case), it still processes each character exactly once. That's the kind of scaling that makes you feel good about your algorithmic life choices.
 
So now we can blast through our 100,000 documents, finding 5,000 keywords in a single pass, catching every overlap and never missing a thing. Pretty solid improvement from where we started.
 
But here's the thing — we still haven't tackled our original autocomplete challenge. We can find exact matches like champs, but what about "Photoshkp" when we're looking for "Photoshop"?

---

## The Fuzzy Matching Problem (Or: Why We're Not Done Yet)

So now we can find exact matches like absolute bosses. But fuzzy matching is a completely different beast.

Here's the problem: with exact matching, we just need to know "does this pattern exist here?" But with fuzzy matching, we need to know "how many characters of this pattern match here?" 

Think about it. If we're looking for "Photoshop" and we see "Photoshkp", we need to count:
- 'P' matches ✓
- 'h' matches ✓  
- 'o' matches ✓
- 't' matches ✓
- 'o' matches ✓
- 's' matches ✓
- 'h' matches ✓
- 'k' ≠ 'o' ✗ (1 typo)
- 'p' matches ✓

So we have 8 out of 9 characters matching. With our "at most 1 typo" rule, this should count as a match.

But here's where things get ugly again. To do this kind of character-by-character counting, we need a **sliding window approach**:

1. Start at position 0 in the text
2. Check how many characters of "Photoshop" match starting here
3. Move to position 1, check again
4. Move to position 2, check again
5. ...and so on for every possible starting position

This means we're back to **O(N × M × V)** complexity where:
- N = text length
- M = pattern length  
- V = number of patterns

Sound familiar? We're basically back to our original nightmare, just with a slightly smarter matching algorithm.

That's when I realized we needed something completely different. We needed to stop thinking about text as a sequence of characters and start thinking about it as... a signal.

---

## Wait, What? Text as a Signal? Have You Lost Your Mind?

Okay, okay, hear me out. I know this sounds absolutely bonkers. 

You're probably thinking: "Signals are for audio and radio waves and stuff. Text is just... text. Letters. Symbols. Not wavy things!"

And you'd be right! But here's the thing — I was sitting there, staring at my sliding window nightmare that was bringing us back to O(N × M × V) complexity, when I had this thought:

*What if we're thinking about this all wrong?*

See, when you search for "photoshop" in a document, what are you really doing? You're looking for a specific pattern: 'p' followed by 'h' followed by 'o'... and so on.

But what if instead of sliding a window and counting matches, we could somehow encode this pattern matching as a mathematical operation that could be computed all at once?

Now imagine I gave you a pair of magical X-ray glasses. When you put them on and look at a document, instead of seeing text, you see something like this:
 
- Everywhere there's a 'p', you see a red dot
- Everywhere there's an 'h', you see a blue dot  
- Everywhere there's an 'o', you see a green dot
- And so on...
 
So the word "photoshop" would look like: 🔴🔵🟢🟡🟢🟣🔵🟢🔴
 
And here's where it gets interesting. What if, instead of dots, we used numbers?
 
- 'p' appears at position 0? Put a 1 there. Doesn't appear? Put a 0.
- 'h' appears at position 1? Put a 1 there. Doesn't appear? Put a 0.
 
Suddenly our text looks like this:
 
```
Text: "photoshop rocks"
'p' signal: [1,0,0,0,0,0,0,0,1,0,0,0,0,0,0]
'h' signal: [0,1,0,0,0,0,1,0,0,0,0,0,0,0,0]
'o' signal: [0,0,1,0,1,0,0,1,0,0,1,0,0,0,0]
```
 
But wait, why would we do this? What's the point?
 
Well, remember how in the naive approach we had to slide our pattern across the text character by character? That's like trying to find Waldo by checking every single person in the crowd one by one.
 
But with signals... oh boy, with signals we can do something magical.
 
## The Magic Trick That Changes Everything
 
Here's a question that'll blow your mind: What if I told you there's a mathematical operation that can check EVERY POSITION IN THE TEXT SIMULTANEOUSLY?
 
*Record scratch* 
 
*Freeze frame*
 
Yup, you heard that right. Instead of:
- Check position 0: "photo..." Nope, not "photoshop"
- Check position 1: "hotos..." Nope, not "photoshop"  
- Check position 2: "otosh..." Nope, not "photoshop"
- ... and so on for 1,000 positions
 
We can literally ask: "Hey math, where does 'photoshop' appear in this text?" 
 
And math goes: "Positions 0, 47, and 892. You're welcome. That'll be O(N log N) operations please."
 
This magical operation? It's called convolution. And the Fast Fourier Transform (FFT) makes it faster than a caffeinated squirrel on Red Bull.
 
But I'm getting ahead of myself. Let me show you why this actually works...
 
## The Secret Sauce: Playing Match Detective with Binary Signals
 
Alright, so we've turned our text into these weird binary signals. Now what?
 
Let me show you something that'll make your brain go "OHHHHH!" 
 
Say we're looking for "cat" in the text "the cat sat". Here's what our signals look like:
 
```
Text: "the cat sat"
Position: 01234567890
 
'c' signal: [0,0,0,0,1,0,0,0,0,0,0]
'a' signal: [0,0,0,0,0,1,0,0,0,1,0]
't' signal: [1,0,0,0,0,0,1,0,0,0,1]
```
 
And our pattern "cat":
```
'c' at position 0: [1,0,0]
'a' at position 1: [0,1,0]
't' at position 2: [0,0,1]
```
 
Now here's the million-dollar question: How do we find where "cat" appears in our text?
 
Well, think about it. The word "cat" appears at position 4 in "the cat sat". What's special about position 4?
 
- At position 4, there's a 'c' (check our 'c' signal - yep, there's a 1 at position 4!)
- At position 5, there's an 'a' (check our 'a' signal - yep, there's a 1 at position 5!)
- At position 6, there's a 't' (check our 't' signal - yep, there's a 1 at position 6!)
 
So if we wanted to check if "cat" starts at position 4, we'd do:
```
c_matches = text_c_signal[4] × pattern_c_signal[0] = 1 × 1 = 1 ✓
a_matches = text_a_signal[5] × pattern_a_signal[1] = 1 × 1 = 1 ✓
t_matches = text_t_signal[6] × pattern_t_signal[2] = 1 × 1 = 1 ✓
 
Total matches = 1 + 1 + 1 = 3 (Perfect! All 3 characters match!)
```
 
But here's the thing - we don't want to check just position 4. We want to check EVERY position. 
 
In the naive approach, we'd do:
- Position 0: Check if "the" matches "cat" (nope)
- Position 1: Check if "he " matches "cat" (nope)
- Position 2: Check if "e c" matches "cat" (nope)
- ... and so on
 
But with our signals, we can do something absolutely bonkers.
 
## The Slide-and-Multiply Trick That Will Melt Your Brain
 
Okay, ready for this? Instead of checking each position one by one, we're going to slide our pattern signal across our text signal and multiply at EVERY position AT THE SAME TIME.
 
*Wait, what?*
 
Let me show you. We'll slide our "cat" pattern across the text and see what happens:
 
<div class="slide-multiply-demo">
    <div class="controls">
        <input type="text" id="slide-text" value="the cat sat" placeholder="Enter text">
        <input type="text" id="slide-pattern" value="cat" placeholder="Enter pattern">
        <button id="slide-animate-btn">Animate Sliding</button>
        <button id="slide-reset-btn">Reset</button>
        <label>Speed: <input type="range" id="slide-speed" min="1" max="10" value="5"></label>
    </div>
    <div id="slide-visualization"></div>
    <div id="slide-explanation"></div>
</div>
 
<script>
let slideMultiplyDemo = {
    text: 'the cat sat',
    pattern: 'cat',
    currentPosition: 0,
    animationId: null,
    speed: 5,
    
    init() {
        document.getElementById('slide-animate-btn').addEventListener('click', () => {
            this.text = document.getElementById('slide-text').value || 'the cat sat';
            this.pattern = document.getElementById('slide-pattern').value || 'cat';
            this.startAnimation();
        });
        
        document.getElementById('slide-reset-btn').addEventListener('click', () => {
            this.reset();
        });
        
        document.getElementById('slide-speed').addEventListener('input', (e) => {
            this.speed = parseInt(e.target.value);
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
        
        // Create character signals
        const uniqueChars = [...new Set((this.text + this.pattern).split(''))].filter(c => c !== ' ');
        const textSignals = {};
        uniqueChars.forEach(char => {
            textSignals[char] = this.text.split('').map(c => c === char ? 1 : 0);
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
            patternSignals[char] = this.pattern.split('').map(c => c === char ? 1 : 0);
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
                const textChar = this.text[this.currentPosition + i];
                const patternChar = this.pattern[i];
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
        
        this.currentPosition = 0;
        let frameCount = 0;
        const framesPerStep = 60 / this.speed;
        
        const animate = () => {
            frameCount++;
            
            if (frameCount >= framesPerStep) {
                frameCount = 0;
                this.currentPosition++;
                this.render();
                
                if (this.currentPosition > this.text.length - this.pattern.length) {
                    this.currentPosition = 0; // Loop back
                }
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
    border: 1px solid var(--border-color);
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
    border: 2px solid var(--border-color);
    border-radius: 6px;
    font-size: 14px;
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
}
 
.position-num {
    width: 30px;
    text-align: center;
    font-size: 10px;
    color: #6b7280;
}
 
.text-char {
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 1px solid var(--border-color);
    border-radius: 4px;
    font-family: 'Courier New', monospace;
    font-weight: bold;
    transition: all 0.3s ease;
}
 
.text-char.highlighted {
    background: #fbbf24;
    border-color: #f59e0b;
    transform: scale(1.1);
}
 
.signal-row {
    display: flex;
    align-items: center;
    margin: 0.5rem 0;
}
 
.char-label {
    width: 40px;
    font-weight: bold;
    font-family: 'Courier New', monospace;
}
 
.signal-values {
    display: flex;
    gap: 4px;
}
 
.signal-bit {
    width: 30px;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    border: 1px solid var(--border-color);
    border-radius: 2px;
    font-size: 12px;
    font-weight: bold;
    transition: all 0.3s ease;
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
    background: #fee2e2;
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
    border: 1px solid var(--border-color);
}
 
.calculation-grid {
    display: flex;
    gap: 1rem;
    margin: 1rem 0;
    flex-wrap: wrap;
}
 
.calc-item {
    padding: 0.5rem;
    background: var(--bg-secondary);
    border-radius: 6px;
    border: 2px solid var(--border-color);
}
 
.calc-item.match {
    border-color: #22c55e;
    background: #d1fae5;
}
 
.calc-item.no-match {
    border-color: #ef4444;
    background: #fee2e2;
}
 
.calc-chars {
    font-family: 'Courier New', monospace;
    font-weight: bold;
    text-align: center;
}
 
.calc-result {
    text-align: center;
    font-weight: bold;
    margin-top: 0.25rem;
}
 
.total-score {
    font-size: 18px;
    font-weight: bold;
    text-align: center;
    margin-top: 1rem;
    padding: 1rem;
    background: var(--bg-secondary);
    border-radius: 8px;
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
    border: 1px solid var(--border-color);
    text-align: center;
}
 
/* Math theorem styling */
.math-theorem {
    margin: 2rem 0;
    padding: 2rem;
    background: #f0f9ff;
    border-left: 4px solid #3b82f6;
    border-radius: 8px;
}
 
.math-theorem h4 {
    margin: 0 0 1rem 0;
    color: #1e40af;
    font-size: 1.2rem;
}
 
.math-theorem p {
    margin: 0.5rem 0;
}
 
.math-theorem em {
    font-family: 'Times New Roman', serif;
    font-size: 1.1em;
    color: #1e40af;
}
 
.math-details {
    margin-top: 1rem;
    padding: 1.5rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
    font-size: 0.95em;
}
 
.math-details code {
    background: var(--bg-primary);
    padding: 0.2rem 0.4rem;
    border-radius: 4px;
}
 
details summary {
    cursor: pointer;
    padding: 0.75rem 1rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
    font-weight: 600;
    transition: all 0.2s ease;
}
 
details summary:hover {
    background: var(--bg-primary);
    border-color: #3b82f6;
}
 
details[open] summary {
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
}
</style>
 
This sliding-and-multiplying operation? It's almost like something called convolution...
 
 
In true convolution, we'd actually need to flip our pattern backwards first. So "cat" would become "tac", then we'd slide that across. But for pattern matching, we don't want the backwards version - we want to find "cat", not "tac"!
 
What we're actually doing is called **cross-correlation**. It's convolution's cooler cousin that doesn't flip things around.
 
But here's the beautiful part: The FFT trick works for both! Whether you're doing convolution (with the flip) or cross-correlation (without the flip), the frequency domain magic still applies. And since everyone just says "convolution" anyway (because "cross-correlation" is a mouthful), we'll keep calling it that. Just know that we're technically doing the no-flip version.
 
Now watch this animation and let it sink in. We're literally sliding our pattern across the text, computing a score at each position. But doing this naively takes O(N × M) time.
 
There has to be a better way, right?
 
*Spoiler: There is. And it involves thinking about our problem in a completely different dimension.*
 
## The Fourier Transform: When Time Meets Frequency
 
Alright, buckle up. We're about to take a detour through one of the most beautiful ideas in mathematics.
 
You know how a musical chord is made up of multiple notes played together? Like, a C major chord is C + E + G all at once? Your ear hears one sound, but it's actually three frequencies mixed together.
 
The Fourier Transform is like having supernatural hearing that can pick apart that chord and tell you exactly which notes are in it. It takes a signal in the "time domain" (what you hear) and shows you what frequencies make it up.
 
But here's the kicker: **The Fourier Transform works on ANY signal. Including our text signals.**
 
Let me show you what I mean. Remember our 'cat' signal?
 
```
Position:  0 1 2 3 4 5 6 7 8 9 10
Text:      t h e   c a t   s a t
'c' signal: 0 0 0 0 1 0 0 0 0 0 0
```
 
This is our signal in the "space domain" (position in text). The Fourier Transform would decompose this into a sum of sine waves of different frequencies. Yes, sine waves. In text. I know it sounds insane, but stay with me.
 
### The Magic Theorem That Changes Everything
 
Here's the theorem that makes computer scientists fall to their knees in worship:
 
<div class="math-theorem">
<h4>The Convolution Theorem</h4>
<p><strong>Statement:</strong> The Fourier transform of a convolution of two signals equals the pointwise product of their Fourier transforms.</p>
<p>In math speak: <em>ℱ{f ∗ g} = ℱ{f} · ℱ{g}</em></p>
<p>In human speak: That horrible sliding-and-multiplying operation becomes simple multiplication after we transform our signals!</p>
</div>
 
Let me break this down:
- **Left side**: Sliding our pattern across text (convolution) = O(N × M) operations
- **Right side**: Transform both, multiply, transform back = O(N log N) operations
 
It's like... imagine you need to multiply 1,234,567 × 8,910,111 in your head. You could:
1. Do it the long way with pencil and paper (like convolution)
2. Or use logarithms: log(a) + log(b) = log(a×b), so just add instead of multiply!
 
The Fourier Transform is doing something similar for our pattern matching.
 
### But Why Does This Work? (For the Brave Souls)
 
<details>
<summary><strong>Click for the FULL mathematical journey (Warning: Serious math ahead!)</strong></summary>
 
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
 
Here's where the magic happens. Let's prove that convolution in time domain equals multiplication in frequency domain.
 
Start with the convolution:
 
$$(f * g)[n] = \sum_{m=0}^{N-1} f[m] \cdot g[n-m]$$
 
Take the DFT of both sides:
 
$$\mathcal{F}\{(f * g)\}[k] = \sum_{n=0}^{N-1} \left(\sum_{m=0}^{N-1} f[m] \cdot g[n-m]\right) \cdot e^{-i2\pi kn/N}$$
 
Swap the sums (we can do this because they're finite):
 
$$= \sum_{m=0}^{N-1} f[m] \sum_{n=0}^{N-1} g[n-m] \cdot e^{-i2\pi kn/N}$$
 
Now here's the clever bit. Let $j = n - m$, so $n = j + m$:
 
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
3. Pointwise multiplication: $O(N)$
4. Inverse FFT: $O(N \log N)$
- Total: $O(N \log N)$
 
The FFT algorithm achieves this speed by using a divide-and-conquer approach, recursively breaking down the DFT computation into smaller subproblems. The implementation details of FFT are beyond the scope of this article, but if you're curious about how it works under the hood, check out this excellent resource: [Fast Fourier Transform on CP-Algorithms](https://cp-algorithms.com/algebra/fft.html).
 
<h4>Part 6: Putting It All Together for Text Matching</h4>
 
For our text matching problem:
 
1. For each character $c$, create binary signals $text_c$ and $pattern_c$
2. Compute $score_c[i] = (text_c \star pattern_c)[i]$ using FFT:
   - $\mathcal{F}\{text_c\}$
   - $\mathcal{F}\{\overline{pattern_c}\}$ (or just conjugate $\mathcal{F}\{pattern_c\}$)
   - Multiply pointwise
   - Inverse FFT
3. Sum over all characters: $score[i] = \sum_{c \in \Sigma} score_c[i]$
4. Positions where $score[i] = |pattern|$ are exact matches!
 
The total complexity is $O(|\Sigma| \cdot N \log N)$ where $|\Sigma|$ is the alphabet size. Since the alphabet is fixed (e.g., 128 for ASCII), this is effectively $O(N \log N)$.
 
<h4>Bonus: The Complex Number Magic</h4>
 
Why do complex numbers make this work? The key is Euler's formula:
 
$$e^{i\theta} = \cos(\theta) + i\sin(\theta)$$
 
So our DFT basis functions $e^{-i2\pi kn/N}$ are actually spinning around the unit circle in the complex plane. Each frequency $k$ spins at a different rate, and the DFT measures how much our signal "resonates" with each spinning frequency.
 
When we multiply in the frequency domain, we're combining these resonances. The inverse FFT then reconstructs the convolution by adding up all these spinning components with the right phases.
 
It's like... imagine you're trying to predict where two spinning dancers will meet. Instead of tracking their every move (convolution), you can:
1. Figure out how fast each is spinning (FFT)
2. Multiply their spin rates (frequency domain multiplication)
3. Calculate where they'll meet (inverse FFT)
 
That's the essence of why this beautiful theorem works!
 
</div>
</details>
 
### OK But Like... Frequencies? In Text? Really?
 
I know, I know. It's weird to think about text having "frequencies". But here's one way to think about it:
 
Imagine you're looking for the pattern "the" in a long document. If "the" appears regularly — say, every 50 characters or so — that's like a rhythm or frequency! The Fourier Transform can detect these patterns.
 
Or think of it this way: Our binary signals (0s and 1s) are just like digital audio samples. And if the Fourier Transform can analyze music, it can analyze our text signals too.
 
### The Fast Part of Fast Fourier Transform
 
Now, computing the Fourier Transform the naive way still takes O(N²) operations. Not much better than our sliding window!
 
But in 1965, Cooley and Tukey rediscovered an algorithm (originally invented by Gauss in 1805!) that computes it in just O(N log N) operations. This is the **Fast** Fourier Transform.
 
The trick? It uses a divide-and-conquer approach, breaking the problem into smaller pieces and cleverly reusing calculations. It's like how merge sort is faster than bubble sort, but for frequency analysis.
 
Here's the process for our text matching:
 
```python
# Pseudocode for FFT-based pattern matching
def fft_pattern_match(text, pattern):
    # 1. Convert to binary signals for each character
    text_signals = create_character_signals(text)
    pattern_signals = create_character_signals(pattern)
    
    # 2. For each character, compute match scores using FFT
    all_scores = []
    for char in alphabet:
        # Transform to frequency domain - O(N log N)
        text_fft = FFT(text_signals[char])
        pattern_fft = FFT(pattern_signals[char])
        
        # Multiply in frequency domain - O(N)
        product = text_fft * pattern_fft
        
        # Transform back - O(N log N)
        char_scores = inverse_FFT(product)
        all_scores.append(char_scores)
    
    # 3. Sum scores across all characters
    total_scores = sum(all_scores)
    
    # 4. Find positions where score equals pattern length
    matches = [i for i, score in enumerate(total_scores) 
               if score == len(pattern)]
    
    return matches
```
 
Total complexity: O(|alphabet| × N log N). For our ASCII text, that's O(128 × N log N), which simplifies to O(N log N) since the alphabet size is constant.
 
For a 1,000-character document:
- Naive sliding: 1,000 × pattern_length operations
- FFT magic: 1,000 × log(1,000) ≈ 10,000 operations
 
That's not just faster — it's algorithmically superior. The gap only gets bigger as your documents get longer!
 
## Show Me The Code!
 
Alright, enough math. Here's the C++ implementation that brings all this theory to life:
 
```cpp
#include <bits/stdc++.h>
using namespace std;
 
// Complex number for FFT
typedef complex<double> cd;
const double PI = acos(-1);
 
// Fast Fourier Transform
void fft(vector<cd>& a, bool invert) {
    int n = a.size();
    if (n == 1) return;
    
    // Bit-reversal permutation
    for (int i = 1, j = 0; i < n; i++) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) {
            j ^= bit;
        }
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }
    
    // Cooley-Tukey FFT
    for (int len = 2; len <= n; len <<= 1) {
        double ang = 2 * PI / len * (invert ? -1 : 1);
        cd wlen(cos(ang), sin(ang));
        for (int i = 0; i < n; i += len) {
            cd w(1);
            for (int j = 0; j < len / 2; j++) {
                cd u = a[i + j];
                cd v = a[i + j + len / 2] * w;
                a[i + j] = u + v;
                a[i + j + len / 2] = u - v;
                w *= wlen;
            }
        }
    }
    
    if (invert) {
        for (cd& x : a) x /= n;
    }
}
 
// Pattern matching using FFT
vector<int> pattern_match_fft(const string& text, const string& pattern) {
    int n = text.length();
    int m = pattern.length();
    
    // Pad to power of 2
    int size = 1;
    while (size < n + m - 1) size <<= 1;
    
    // Result will store match scores at each position
    vector<double> result(n, 0);
    
    // Process each character separately
    for (char ch = 'a'; ch <= 'z'; ch++) {
        vector<cd> text_signal(size), pattern_signal(size);
        
        // Create binary signals
        for (int i = 0; i < n; i++) {
            text_signal[i] = (text[i] == ch) ? cd(1, 0) : cd(0, 0);
        }
        
        // Reverse pattern for cross-correlation
        for (int i = 0; i < m; i++) {
            pattern_signal[i] = (pattern[m - 1 - i] == ch) ? cd(1, 0) : cd(0, 0);
        }
        
        // FFT both signals
        fft(text_signal, false);
        fft(pattern_signal, false);
        
        // Multiply in frequency domain
        for (int i = 0; i < size; i++) {
            text_signal[i] *= pattern_signal[i];
        }
        
        // Inverse FFT
        fft(text_signal, true);
        
        // Add to result
        for (int i = 0; i < n - m + 1; i++) {
            result[i] += real(text_signal[i + m - 1]);
        }
    }
    
    // Find matches (score = pattern length)
    vector<int> matches;
    for (int i = 0; i < n - m + 1; i++) {
        if (abs(result[i] - m) < 1e-9) {  // Floating point comparison
            matches.push_back(i);
        }
    }
    
    return matches;
}
 
// For fuzzy matching - find positions where score >= threshold
vector<pair<int, int>> fuzzy_match_fft(const string& text, 
                                       const string& pattern, 
                                       int max_mismatches = 1) {
    int n = text.length();
    int m = pattern.length();
    int threshold = m - max_mismatches;
    
    // Same setup as exact matching
    int size = 1;
    while (size < n + m - 1) size <<= 1;
    
    vector<double> result(n, 0);
    
    // Process each character
    for (char ch = 'a'; ch <= 'z'; ch++) {
        vector<cd> text_signal(size), pattern_signal(size);
        
        for (int i = 0; i < n; i++) {
            text_signal[i] = (text[i] == ch) ? cd(1, 0) : cd(0, 0);
        }
        
        for (int i = 0; i < m; i++) {
            pattern_signal[i] = (pattern[m - 1 - i] == ch) ? cd(1, 0) : cd(0, 0);
        }
        
        fft(text_signal, false);
        fft(pattern_signal, false);
        
        for (int i = 0; i < size; i++) {
            text_signal[i] *= pattern_signal[i];
        }
        
        fft(text_signal, true);
        
        for (int i = 0; i < n - m + 1; i++) {
            result[i] += real(text_signal[i + m - 1]);
        }
    }
    
    // Find fuzzy matches
    vector<pair<int, int>> fuzzy_matches;
    for (int i = 0; i < n - m + 1; i++) {
        int score = round(result[i]);
        if (score >= threshold) {
            fuzzy_matches.push_back({i, score});  // position, match_score
        }
    }
    
    return fuzzy_matches;
}
 
int main() {
    string text = "the cat sat on the mat";
    string pattern = "cat";
    
    // Exact matching
    vector<int> matches = pattern_match_fft(text, pattern);
    cout << "Exact matches at positions: ";
    for (int pos : matches) {
        cout << pos << " ";
    }
    cout << "\n";
    
    // Fuzzy matching (1 mismatch allowed)
    string fuzzy_pattern = "cxt";  // 'cat' with one typo
    auto fuzzy_matches = fuzzy_match_fft(text, fuzzy_pattern, 1);
    cout << "Fuzzy matches: ";
    for (auto [pos, score] : fuzzy_matches) {
        cout << "pos=" << pos << " (score=" << score << ") ";
    }
    cout << "\n";
    
    return 0;
}
```
 
The beauty of this approach? **That score vector we compute is exactly what we need for fuzzy matching!** Instead of checking if `score == pattern_length`, we check if `score >= pattern_length - allowed_typos`. 
 
Want to allow 2 typos? Just check for `score >= pattern_length - 2`. It's that simple.
 
## See The FFT Magic In Action!
 
Now let's visualize exactly what happens when we use FFT for pattern matching. This demo shows you the complete transformation from text to score vector:
 
<div class="fft-visualization-demo">
    <div class="controls">
        <div class="input-group">
            <label>Text:</label>
            <input type="text" id="fft-text" value="the cat sat on the mat" placeholder="Enter text">
        </div>
        <div class="input-group">
            <label>Pattern:</label>
            <input type="text" id="fft-pattern" value="cat" placeholder="Enter pattern">
        </div>
        <button id="fft-compute-btn" class="primary-btn">Compute Score Vector 🚀</button>
    </div>
    
    <div id="fft-process">
        <!-- Will be populated by JavaScript -->
    </div>
</div>
 
<script>
class FFTPatternMatcher {
    constructor() {
        this.text = '';
        this.pattern = '';
    }
    
    // Simple DFT for demonstration (not optimized FFT)
    dft(signal) {
        const N = signal.length;
        const result = [];
        
        for (let k = 0; k < N; k++) {
            let real = 0, imag = 0;
            
            for (let n = 0; n < N; n++) {
                const angle = -2 * Math.PI * k * n / N;
                real += signal[n] * Math.cos(angle);
                imag += signal[n] * Math.sin(angle);
            }
            
            result.push({ real, imag, magnitude: Math.sqrt(real*real + imag*imag) });
        }
        
        return result;
    }
    
    // Inverse DFT
    idft(spectrum) {
        const N = spectrum.length;
        const result = [];
        
        for (let n = 0; n < N; n++) {
            let real = 0;
            
            for (let k = 0; k < N; k++) {
                const angle = 2 * Math.PI * k * n / N;
                real += spectrum[k].real * Math.cos(angle) - spectrum[k].imag * Math.sin(angle);
            }
            
            result.push(real / N);
        }
        
        return result;
    }
    
    // Pad signal to power of 2
    padSignal(signal, targetLength) {
        const padded = [...signal];
        while (padded.length < targetLength) {
            padded.push(0);
        }
        return padded;
    }
    
    compute() {
        const container = document.getElementById('fft-process');
        
        // Get unique characters
        const chars = [...new Set(this.text + this.pattern)].filter(c => c !== ' ').sort();
        
        // Pad to power of 2
        const fftLength = Math.pow(2, Math.ceil(Math.log2(this.text.length)));
        
        let html = '';
        
        // Step 1: Show original text and pattern
        html += '<div class="step-block">';
        html += '<h3>Input</h3>';
        html += `<div class="text-display">Text: <span class="mono">"${this.text}"</span></div>`;
        html += `<div class="text-display">Pattern: <span class="mono">"${this.pattern}"</span></div>`;
        html += '</div>';
        
        // Step 2: Binary signals
        html += '<div class="step-block">';
        html += '<h3>Step 1: Convert to Binary Signals</h3>';
        html += '<div class="signals-grid">';
        
        const signals = {};
        chars.forEach(char => {
            const textSignal = this.text.split('').map(c => c === char ? 1 : 0);
            const patternSignal = this.pattern.split('').map(c => c === char ? 1 : 0);
            
            html += `<div class="char-signals">`;
            html += `<div class="char-header">Character '${char}'</div>`;
            html += `<div class="signal-row">`;
            html += `<span class="signal-label">Text:</span>`;
            html += `<span class="signal-bits">`;
            textSignal.forEach((bit, i) => {
                html += `<span class="bit ${bit ? 'one' : 'zero'}">${bit}</span>`;
            });
            html += `</span>`;
            html += `</div>`;
            html += `<div class="signal-row">`;
            html += `<span class="signal-label">Pattern:</span>`;
            html += `<span class="signal-bits">`;
            patternSignal.forEach((bit, i) => {
                html += `<span class="bit ${bit ? 'one' : 'zero'}">${bit}</span>`;
            });
            html += ` (reversed for correlation)`;
            html += `</span>`;
            html += `</div>`;
            html += `</div>`;
            
            signals[char] = {
                text: this.padSignal(textSignal, fftLength),
                pattern: this.padSignal([...patternSignal].reverse(), fftLength)
            };
        });
        html += '</div>';
        html += '</div>';
        
        // Step 3: FFT visualization
        html += '<div class="step-block">';
        html += '<h3>Step 2: Apply FFT (Transform to Frequency Domain)</h3>';
        html += '<div class="fft-visual">';
        html += '<p>Converting signals to frequency domain using FFT...</p>';
        html += '<div class="transform-animation">';
        html += '<span class="transform-box">Binary Signals</span>';
        html += '<span class="arrow">→ FFT →</span>';
        html += '<span class="transform-box">Frequency Domain</span>';
        html += '</div>';
        
        // Show frequency magnitudes for first character
        if (chars.length > 0) {
            const char = chars[0];
            const textFFT = this.dft(signals[char].text);
            const patternFFT = this.dft(signals[char].pattern);
            
            html += `<div class="freq-display">`;
            html += `<p>Example: Frequency magnitudes for character '${char}'</p>`;
            html += `<div class="freq-chart">`;
            
            // Show first 8 frequencies
            for (let i = 0; i < Math.min(8, textFFT.length); i++) {
                const textMag = textFFT[i].magnitude;
                const patternMag = patternFFT[i].magnitude;
                const maxMag = Math.max(textMag, patternMag, 1);
                
                html += `<div class="freq-pair">`;
                html += `<div class="freq-label">f${i}</div>`;
                html += `<div class="freq-bars">`;
                html += `<div class="freq-bar text" style="height: ${(textMag/maxMag)*50}px" title="Text: ${textMag.toFixed(2)}"></div>`;
                html += `<div class="freq-bar pattern" style="height: ${(patternMag/maxMag)*50}px" title="Pattern: ${patternMag.toFixed(2)}"></div>`;
                html += `</div>`;
                html += `</div>`;
            }
            
            html += `</div>`;
            html += `</div>`;
        }
        
        html += '</div>';
        html += '</div>';
        
        // Step 4: Multiplication
        html += '<div class="step-block">';
        html += '<h3>Step 3: Multiply in Frequency Domain</h3>';
        html += '<div class="multiply-visual">';
        html += '<p>For each character, multiply corresponding frequency components:</p>';
        html += '<div class="equation-display">';
        html += 'FFT(text) × FFT(pattern) = FFT(correlation)';
        html += '</div>';
        html += '<p class="note">This multiplication in frequency domain equals correlation in time domain!</p>';
        html += '</div>';
        html += '</div>';
        
        // Step 5: IFFT and Score Vector
        html += '<div class="step-block">';
        html += '<h3>Step 4: Apply Inverse FFT to Get Score Vector</h3>';
        
        // Calculate actual scores
        const scores = new Array(this.text.length).fill(0);
        chars.forEach(char => {
            const textFFT = this.dft(signals[char].text);
            const patternFFT = this.dft(signals[char].pattern);
            
            // Multiply in frequency domain
            const product = textFFT.map((t, i) => ({
                real: t.real * patternFFT[i].real - t.imag * patternFFT[i].imag,
                imag: t.real * patternFFT[i].imag + t.imag * patternFFT[i].real
            }));
            
            // IFFT
            const correlation = this.idft(product);
            
            // Add to scores
            for (let i = 0; i < this.text.length - this.pattern.length + 1; i++) {
                scores[i] += Math.round(correlation[i + this.pattern.length - 1]);
            }
        });
        
        html += '<div class="score-vector">';
        html += '<h4>Final Score Vector:</h4>';
        html += '<div class="score-display">';
        
        // Display text with positions
        html += '<div class="text-positions">';
        for (let i = 0; i < this.text.length; i++) {
            html += `<span class="char-pos">${this.text[i]}</span>`;
        }
        html += '</div>';
        
        // Display position numbers
        html += '<div class="position-numbers">';
        for (let i = 0; i < this.text.length; i++) {
            html += `<span class="pos-num">${i}</span>`;
        }
        html += '</div>';
        
        // Display scores
        html += '<div class="scores">';
        for (let i = 0; i < this.text.length - this.pattern.length + 1; i++) {
            const score = scores[i];
            const isMatch = score === this.pattern.length;
            const isClose = score >= this.pattern.length - 1;
            
            html += `<span class="score-value ${isMatch ? 'match' : isClose ? 'close' : ''}">${score}</span>`;
        }
        html += '</div>';
        
        html += '</div>';
        
        // Explanation
        html += '<div class="score-explanation">';
        html += '<h4>What does this mean?</h4>';
        html += '<ul>';
        html += `<li><strong>Score = ${this.pattern.length}</strong>: Perfect match! All characters match.</li>`;
        html += `<li><strong>Score = ${this.pattern.length - 1}</strong>: Fuzzy match with 1 character different.</li>`;
        html += `<li><strong>Score = ${this.pattern.length - 2}</strong>: Fuzzy match with 2 characters different.</li>`;
        html += '</ul>';
        
        // Highlight matches
        const matches = [];
        for (let i = 0; i < scores.length; i++) {
            if (scores[i] === this.pattern.length) {
                matches.push(i);
            }
        }
        
        if (matches.length > 0) {
            html += `<p class="match-found">✅ Found exact matches at position(s): ${matches.join(', ')}</p>`;
        }
        
        // Show fuzzy matches
        const fuzzyMatches = [];
        for (let i = 0; i < scores.length; i++) {
            if (scores[i] === this.pattern.length - 1) {
                fuzzyMatches.push(i);
            }
        }
        
        if (fuzzyMatches.length > 0) {
            html += `<p class="fuzzy-found">🔍 Found fuzzy matches (1 typo) at position(s): ${fuzzyMatches.join(', ')}</p>`;
        }
        
        html += '</div>';
        html += '</div>';
        html += '</div>';
        
        container.innerHTML = html;
    }
}
 
// Initialize
document.addEventListener('DOMContentLoaded', function() {
    const matcher = new FFTPatternMatcher();
    
    document.getElementById('fft-compute-btn').addEventListener('click', function() {
        matcher.text = document.getElementById('fft-text').value.toLowerCase();
        matcher.pattern = document.getElementById('fft-pattern').value.toLowerCase();
        
        if (matcher.text && matcher.pattern) {
            matcher.compute();
        }
    });
});
</script>
 
<style>
.fft-visualization-demo {
    margin: 2rem 0;
    padding: 2rem;
    background: var(--bg-secondary);
    border-radius: 12px;
    border: 1px solid var(--border-color);
}
 
.controls {
    display: flex;
    gap: 1rem;
    align-items: flex-end;
    margin-bottom: 2rem;
    flex-wrap: wrap;
}
 
.input-group {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
}
 
.input-group label {
    font-weight: 600;
    font-size: 14px;
}
 
.input-group input {
    padding: 0.5rem 1rem;
    border: 2px solid var(--border-color);
    border-radius: 6px;
    font-size: 14px;
    min-width: 200px;
}
 
.primary-btn {
    padding: 0.75rem 1.5rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 600;
    font-size: 16px;
    cursor: pointer;
    transition: all 0.2s ease;
}
 
.primary-btn:hover {
    background: #2563eb;
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}
 
.secondary-btn {
    padding: 0.75rem 1.5rem;
    background: transparent;
    color: #3b82f6;
    border: 2px solid #3b82f6;
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
}
 
.secondary-btn:hover {
    background: #3b82f6;
    color: white;
}
 
.fft-stage {
    margin: 2rem 0;
    padding: 1.5rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
}
 
.fft-stage h4 {
    margin: 0 0 1rem 0;
    color: #1f2937;
    font-size: 1.1rem;
}
 
/* Binary signals */
.char-signal-row {
    display: flex;
    gap: 2rem;
    align-items: center;
    margin: 0.5rem 0;
    padding: 0.5rem;
    background: var(--bg-secondary);
    border-radius: 6px;
}
 
.signal-display {
    display: flex;
    align-items: center;
    gap: 1rem;
}
 
.signal-label {
    font-size: 12px;
    font-weight: 600;
    color: #6b7280;
    min-width: 50px;
}
 
.signal-values {
    display: flex;
    gap: 3px;
}
 
.bit {
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #f3f4f6;
    border: 1px solid #e5e7eb;
    border-radius: 3px;
    font-size: 11px;
    font-weight: 600;
}
 
.bit.active {
    background: #3b82f6;
    color: white;
    border-color: #3b82f6;
}
 
/* FFT visualization */
.spectrum-view {
    margin: 1rem 0;
}
 
.spectrum-row {
    display: flex;
    align-items: flex-end;
    gap: 1rem;
    margin: 1rem 0;
}
 
.spectrum-label {
    font-size: 12px;
    font-weight: 600;
    min-width: 100px;
}
 
.spectrum-bars {
    display: flex;
    gap: 2px;
    align-items: flex-end;
    height: 100px;
}
 
.freq-bar {
    width: 20px;
    background: #3b82f6;
    border-radius: 2px 2px 0 0;
    transition: all 0.3s ease;
    cursor: help;
}
 
.freq-bar.pattern {
    background: #10b981;
}
 
.freq-bar:hover {
    opacity: 0.8;
}
 
/* Multiplication visualization */
.multiply-visual {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 2rem;
    margin: 2rem 0;
    font-size: 3rem;
}
 
.equation {
    text-align: center;
    font-family: 'Courier New', monospace;
    font-size: 1.2rem;
    margin: 1rem 0;
    padding: 1rem;
    background: var(--bg-secondary);
    border-radius: 6px;
}
 
/* Scores */
.char-scores {
    margin: 0.5rem 0;
}
 
.score-values {
    display: flex;
    gap: 4px;
    margin-top: 0.5rem;
}
 
.score {
    width: 25px;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #f3f4f6;
    border: 1px solid #e5e7eb;
    border-radius: 3px;
    font-size: 11px;
    font-weight: 600;
}
 
.score.positive {
    background: #d1fae5;
    border-color: #10b981;
    color: #059669;
}
 
/* Final result */
.text-result {
    font-family: 'Courier New', monospace;
    font-size: 1.5rem;
    text-align: center;
    margin: 2rem 0;
    letter-spacing: 0.1em;
}
 
.match-highlight {
    background: #fbbf24;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-weight: bold;
}
 
.score-bars {
    display: flex;
    gap: 4px;
    align-items: flex-end;
    justify-content: center;
    margin: 1rem 0;
    height: 100px;
}
 
.score-bar {
    width: 30px;
    background: #e5e7eb;
    border-radius: 4px 4px 0 0;
    position: relative;
    transition: all 0.3s ease;
}
 
.score-bar.match {
    background: #10b981;
}
 
.score-value {
    position: absolute;
    top: -20px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 11px;
    font-weight: bold;
}
 
.position {
    position: absolute;
    bottom: -20px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 10px;
    color: #6b7280;
}
 
/* Explanation */
.explanation-box {
    margin-top: 2rem;
    padding: 1.5rem;
    background: var(--bg-primary);
    border-radius: 8px;
    border: 1px solid var(--border-color);
}
 
.explanation-box h5 {
    margin: 0 0 1rem 0;
    color: #1f2937;
}
 
.explanation-box ol {
    margin: 1rem 0;
    padding-left: 1.5rem;
}
 
.explanation-box .success {
    color: #059669;
    font-weight: 600;
    margin: 1rem 0;
}
 
.explanation-box .info {
    color: #3b82f6;
    margin: 1rem 0;
}
 
.explanation-box .complexity {
    font-size: 14px;
    color: #6b7280;
    font-style: italic;
}
 
.note {
    font-size: 12px;
    color: #6b7280;
    font-style: italic;
    text-align: center;
    margin-top: 1rem;
}
</style>
 
---