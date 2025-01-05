# ELearning-Unpam
Auto Answer Elearning

```
// Token bucket rate limiter
class RateLimiter {
    constructor(tokensPerMinute) {
        this.maxTokens = tokensPerMinute;
        this.tokens = tokensPerMinute;
        this.lastRefill = Date.now();
        this.tokensPerMs = tokensPerMinute / (60 * 1000);
    }

    async waitForToken() {
        this.refillTokens();
        if (this.tokens < 1) {
            const waitTime = (1 - this.tokens) / this.tokensPerMs;
            console.log(`Rate limit reached. Waiting ${Math.ceil(waitTime / 1000)} seconds...`);
            await new Promise(resolve => setTimeout(resolve, waitTime));
            this.refillTokens();
        }
        this.tokens -= 1;
        return true;
    }

    refillTokens() {
        const now = Date.now();
        const timePassed = now - this.lastRefill;
        this.tokens = Math.min(this.maxTokens, this.tokens + timePassed * this.tokensPerMs);
        this.lastRefill = now;
    }
}

const rateLimiter = new RateLimiter(20); // Increased to 20 requests per minute for Flash model

async function fetchAnswerFromGeminiAI(question) {
    const apiKey = 'YOUR_API_KEY';
    // Changed to use the Flash model instead of Pro
    const apiUrl = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent';
    const maxRetries = 5;
    const baseDelay = 3000; // Reduced delay for Flash model
    
    const enhancedQuestion = `jawab dengan jawaban paling ringkas dan mudah dimengerti, gunakan bahasa sebagai seorang mahasiswa, tidak perlu menggunakan referensi. Berikut adalah konteks atau pertanyaan yang saling berhubungan, jangan menjawab menggunakan bold, garis miring atau text style yang lainnya, hanya memperbolehkan point, atau numbering, jadi semua jenis font normal, jawabannya harus panjang dan jelas: ${question}`;
    
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            await rateLimiter.waitForToken();

            const response = await fetch(`${apiUrl}?key=${apiKey}`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    contents: [{
                        parts: [{
                            text: enhancedQuestion
                        }]
                    }]
                })
            });

            if (response.status === 429) {
                const delay = baseDelay * Math.pow(2, attempt);
                console.log(`Rate limited. Waiting ${delay/1000} seconds before retry ${attempt + 1}/${maxRetries}`);
                await new Promise(resolve => setTimeout(resolve, delay));
                continue;
            }

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const data = await response.json();
            if (data?.candidates?.[0]?.content?.parts?.[0]?.text) {
                return data.candidates[0].content.parts[0].text;
            } else {
                throw new Error('Invalid response structure');
            }
        } catch (error) {
            if (attempt === maxRetries - 1) {
                throw error;
            }
            console.warn(`Attempt ${attempt + 1} failed:`, error);
            await new Promise(resolve => setTimeout(resolve, baseDelay * Math.pow(2, attempt)));
        }
    }
    
    throw new Error('Max retries exceeded');
}

async function processPost(post, waitForElement) {
    const contentContainer = post.querySelector('.post-content-container');
    if (!contentContainer) return;

    let allContent = Array.from(contentContainer.querySelectorAll('p, p span'))
        .map(el => el.textContent.trim())
        .join(' ')
        .replace(/\s+/g, ' ')
        .trim();

    if (!allContent) return;

    let textarea = post.querySelector('textarea[name="post"]');
    if (!textarea) {
        const replyButton = post.querySelector('a[data-action="collapsible-link"]');
        if (replyButton) {
            replyButton.click();
            textarea = await waitForElement('textarea[name="post"]', post);
        }
    }

    if (!textarea) return;

    try {
        textarea.value = 'Sedang mengambil jawaban...';
        const answer = await fetchAnswerFromGeminiAI(allContent);
        textarea.value = answer;
        textarea.dispatchEvent(new Event('input', { bubbles: true }));

        await delay(2000); // Reduced delay

        const submitButton = post.querySelector('button[data-region="submit-text"]')?.closest('button') ||
            Array.from(post.querySelectorAll('button'))
                .find(button => button.textContent.trim().toLowerCase() === 'post to forum');

        if (submitButton) {
            submitButton.click();
        }
    } catch (error) {
        console.error('Error processing post:', error);
        textarea.value = `Maaf, terjadi kesalahan: ${error.message}`;
    }
}

async function autoAnswerQuestions() {
    const waitForElement = (selector, parent, timeout = 5000) => {
        return new Promise((resolve) => {
            if (parent.querySelector(selector)) {
                return resolve(parent.querySelector(selector));
            }

            const observer = new MutationObserver(() => {
                if (parent.querySelector(selector)) {
                    observer.disconnect();
                    resolve(parent.querySelector(selector));
                }
            });

            observer.observe(parent, {
                childList: true,
                subtree: true
            });

            setTimeout(() => {
                observer.disconnect();
                resolve(null);
            }, timeout);
        });
    };

    const posts = document.querySelectorAll('.forumpost');
    for (const post of posts) {
        await processPost(post, waitForElement);
        await delay(3000); // Reduced delay between posts
    }
}

function delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

// Start automatically
autoAnswerQuestions();
```
