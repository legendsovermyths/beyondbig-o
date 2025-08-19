---
layout: post
title: "Autocomplete on Steroids: Finding Thousands of Keywords in Millions of Documents (with Fuzzy Matching)"
date: 2024-03-15
author: "Anirudh Singh"
excerpt: "How text search turns into automata theory and signal processing when your boss asks for the impossible. A journey from Y2K-era document processing nightmares to mathematical wizardry."
tags: ["algorithms", "automata-theory", "fft", "text-processing", "aho-corasick", "interactive"]
reading_time: 25
---

So there I was, sitting in my beige cubicle in the year 2000, surrounded by towers of CDs containing millions of user documents. Y2K didn't kill us, but this problem might.

Someone upstairs had this brilliant idea: "Hey, let's search through all these documents for software product mentions!" Sounds simple enough, right? Just take our list of 10,000 keywords — every Adobe product that's ever existed, Microsoft Office variations, programming languages, you name it — and find them in our document collection.

But then they hit me with the plot twist: "Oh, and people make typos since autocorrect isn't a thing yet, so catch those too." 

Ah yes, because life wasn't complicated enough already. So now "Photoshop" needs to match "Photoshp", "Ilustrator" needs to catch "Illustrator", and basically any word with at most 1 character wrong.

I'm staring at my 1GHz Pentium III thinking, "This is fine. Everything is fine." Time for some math that's definitely going to ruin my day...

- **10,000 keywords** to search for
- **1,000,000+ documents** to search through  
- **Average document length**: ~500 characters
- **Fuzzy matching**: Every keyword needs to match variants with 1 typo

So "Photoshop" now needs to match "Photoshop", "Phtoshop", "Photosho", "Xhotoshop", "Photoshox"... and about 5,000 other variations. Cool cool cool.

Obviously, my first thought was, "I'll just check every keyword against every document. What could possibly go wrong?"

**10,000 keywords × ~5,000 typo variants × 1,000,000 documents = 50 billion string comparisons**

My poor Pentium would still be crunching numbers when humans colonize Mars.

But here's where it gets interesting — turns out you can actually solve this nightmare with some automata theory and signal processing wizardry. I know, I know, that sounds like I'm just throwing fancy words around, but stick with me here.

---

## The "Obvious" Solution

Alright, so first things first — let me show you what I tried initially, because I'm sure you're thinking the same thing.

Just loop through every document, then loop through every keyword, and check if it's there. Classic nested loops. Your CS professor would be proud.

```cpp
vector<pair<string, string>> search_documents_naive(
    const vector<string>& documents, 
    const vector<string>& keywords) {
    
    vector<pair<string, string>> results;
    
    for (const auto& doc : documents) {        // 1,000,000 documents
        for (const auto& keyword : keywords) { // 10,000 keywords
            if (doc.find(keyword) != string::npos) { // String search
                results.emplace_back(doc, keyword);
            }
        }
    }
    return results;
}
```

So that's **O(D × K × N)** complexity. Plugging in our numbers: 1,000,000 × 10,000 × 500 = 5 trillion operations.

My Pentium III would take about 3 months to finish. No big deal, right? I'll just grab some coffee and wait.

Oh wait, I almost forgot about the fuzzy matching part. *nervous laughter* So now every keyword needs to generate all its possible typo variations first.

```cpp
vector<string> generate_typos(const string& word) {
    vector<string> variations;
    variations.push_back(word);  // Original
    
    // Deletions: "photoshop" -> "hotoshop", "ptoshop", etc.
    for (size_t i = 0; i < word.length(); ++i) {
        variations.push_back(word.substr(0, i) + word.substr(i + 1));
    }
    
    // Substitutions: "photoshop" -> "ahotoshop", "bhotoshop", etc.
    for (size_t i = 0; i < word.length(); ++i) {
        for (char c = 'a'; c <= 'z'; ++c) {
            if (c != word[i]) {
                string variant = word;
                variant[i] = c;
                variations.push_back(variant);
            }
        }
    }
    
    // Insertions: "photoshop" -> "aphotoshop", "pahotoshop", etc.
    for (size_t i = 0; i <= word.length(); ++i) {
        for (char c = 'a'; c <= 'z'; ++c) {
            variations.push_back(word.substr(0, i) + c + word.substr(i));
        }
    }
    
    return variations;
}
```

Now we're looking at **O(D × K × V × N)** where V is about 5,000 variations per keyword.

1,000,000 × 10,000 × 5,000 × 500 = 25 quadrillion operations.

Yeah, that's not happening. At this point my computer would finish sometime around when the sun expands and consumes the Earth.

Here, let me show you just how spectacularly this approach fails:

<div class="naive-demo">
    <div id="naive-visualization"></div>
    <div class="demo-controls">
        <label>Documents: <span id="doc-count">1000</span></label>
        <input type="range" id="doc-slider" min="100" max="10000" value="1000" step="100">
        
        <label>Keywords: <span id="keyword-count">100</span></label>
        <input type="range" id="keyword-slider" min="10" max="1000" value="100" step="10">
        
        <label>Fuzzy matching: <input type="checkbox" id="fuzzy-toggle" checked></label>
    </div>
    <div id="computation-stats"></div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js"></script>
<script>
let naiveSketch = function(p) {
    let docCount = 1000;
    let keywordCount = 100;
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
        let variations = fuzzyEnabled ? 5000 : 1;
        let totalOps = docCount * keywordCount * variations * 500; // 500 chars avg
        
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

Try turning on fuzzy matching in that demo and watch the numbers go absolutely mental. Each red square is basically your CPU having a small existential crisis.

Even with just 1,000 documents and 100 keywords, we're already in the millions of operations. Scale that up to real numbers and you can see why this approach is about as effective as trying to empty the ocean with a teaspoon.

---

## Enter the Trie (And Why It's Actually Genius)

So I'm sitting there, watching my CPU slowly die, when I remembered something from competitive programming. There's this data structure called a trie — basically a tree where each path from root to leaf spells out a word.

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

So now we can walk through a document character by character, following paths in our trie. If a path exists for the current character, great! If not, back to the root we go. Simple, right?

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

So we're not actually done with those 5 characters. We need to start over from position 2 and check "bcdf", then position 3 and check "cdf", then position 4... you get the idea.

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

Try the step-through mode above — it's actually pretty wild to watch. You can see exactly when the failure links kick in and how it manages to find overlapping matches that a basic trie would totally miss.

So what just happened here? The Aho-Corasick automaton solved the fundamental problem with our trie approach. Instead of restarting from scratch every time we hit a dead end, those failure links let us maintain all the progress we've made.

When we're processing "she" and looking for both "he" and "she", the failure links ensure we're simultaneously tracking both possibilities. No more missed overlaps, no more multiple passes through the same text.

The complexity ends up being **O(n + m + z)** where n is text length, m is total pattern length, and z is the number of matches. Compare that to:
- Our original nightmare: O(D × K × V × N) = 25 quadrillion operations  
- Basic trie: O(n + m) but misses stuff
- Aho-Corasick: O(n + m + z) and catches everything

Whether you've got 10 patterns or 10,000 patterns, it still processes each character exactly once. That's the kind of scaling that makes you feel good about your life choices.

So now we can blast through our millions of documents, finding thousands of keywords in a single pass, catching every overlap and never missing a thing. Pretty solid improvement from where we started.

But here's the thing — we still haven't tackled our original Y2K nightmare. We can find exact matches like champs, but what about "Photoshp" when we're looking for "Photoshop"?

This is where we leave the comfortable world of tries and automata behind and venture into something completely different. We're about to treat text like a signal and throw some serious math at it.

---
