# অ্যাক্সেস টোকেন এবং রিফ্রেশ টোকেন: সম্পূর্ণ গাইড

## ১. অ্যাক্সেস টোকেন এবং রিফ্রেশ টোকেন কি?

- **অ্যাক্সেস টোকেন:** এটি একটি স্বল্পমেয়াদী টোকেন যা ব্যবহারকারীর অথেন্টিকেশন এবং অথোরাইজেশন প্রমাণ করে। এটি সাধারণত JWT (JSON Web Token) ফরম্যাটে থাকে এবং এর মেয়াদ কয়েক মিনিট থেকে কয়েক ঘন্টা পর্যন্ত হয়।
- **রিফ্রেশ টোকেন:** এটি একটি দীর্ঘমেয়াদী টোকেন যা নতুন অ্যাক্সেস টোকেন পেতে ব্যবহৃত হয়। এর মেয়াদ কয়েক দিন থেকে কয়েক মাস পর্যন্ত হয়।

## ২. কেন অ্যাক্সেস টোকেন এবং রিফ্রেশ টোকেন ব্যবহার করি?

- **নিরাপত্তা:** অ্যাক্সেস টোকেনের মেয়াদ সীমিত হওয়ায়, যদি টোকেন চুরি হয়, তাহলে তা দীর্ঘ সময় ব্যবহার করা যাবে না।
- **ব্যবহারকারীর অভিজ্ঞতা:** ব্যবহারকারীকে বারবার লগইন করতে হয় না, কারণ রিফ্রেশ টোকেন ব্যবহার করে নতুন অ্যাক্সেস টোকেন জেনারেট করা যায়।
- **স্কেলেবিলিটি:** অ্যাক্সেস টোকেনের মাধ্যমে সার্ভারকে প্রতিটি রিকোয়েস্টে ডাটাবেস চেক করতে হয় না, যা পারফরম্যান্স উন্নত করে।

## ৩. কিভাবে অ্যাক্সেস টোকেন এবং রিফ্রেশ টোকেন কাজ করে?

1. **লগইন বা রেজিস্ট্রেশন:** ব্যবহারকারী লগইন বা রেজিস্ট্রেশন করে।
2. **টোকেন জেনারেট করা:** সার্ভার একটি অ্যাক্সেস টোকেন এবং একটি রিফ্রেশ টোকেন তৈরি করে।
3. **টোকেন ক্লায়েন্টে পাঠানো:** অ্যাক্সেস টোকেন রেসপন্স বডিতে এবং রিফ্রেশ টোকেন HTTP-Only কুকি হিসেবে পাঠানো হয়।
4. **অ্যাক্সেস টোকেন ব্যবহার করে API কল:** ক্লায়েন্ট প্রতিটি API রিকোয়েস্টে অ্যাক্সেস টোকেনটি হেডারে যোগ করে।
5. **অ্যাক্সেস টোকেনের মেয়াদ শেষ হলে:** ক্লায়েন্ট রিফ্রেশ টোকেন ব্যবহার করে নতুন অ্যাক্সেস টোকেন রিকোয়েস্ট করে।
6. **রিফ্রেশ টোকেনের মেয়াদ শেষ হলে:** ব্যবহারকারীকে আবার লগইন করতে হবে।

## ৪. Node.js এ ইমপ্লিমেন্টেশন

### টোকেন জেনারেট করা:
```javascript
const jwt = require('jsonwebtoken');

const generateAccessToken = (user) => {
    return jwt.sign({ id: user.id, role: user.role }, 'your-secret-key', { expiresIn: '15m' });
};

const generateRefreshToken = (user) => {
    return jwt.sign({ id: user.id }, 'your-refresh-secret-key', { expiresIn: '7d' });
};

#login route
app.post('/login', (req, res) => {
    const { email, password } = req.body;

    // Check user credentials (pseudo-code)
    const user = db.findUserByEmail(email);
    if (!user || user.password !== password) {
        return res.status(401).json({ message: 'Invalid credentials' });
    }

    // Generate tokens
    const accessToken = generateAccessToken(user);
    const refreshToken = generateRefreshToken(user);

    // Save refresh token in database
    db.saveRefreshToken(user.id, refreshToken);

    // Send tokens to client
    res.cookie('refreshToken', refreshToken, { httpOnly: true, secure: true });
    res.json({ accessToken });
});

#Refresh token api

app.post('/refresh-token', (req, res) => {
    const refreshToken = req.cookies.refreshToken;

    // Verify refresh token
    jwt.verify(refreshToken, 'your-refresh-secret-key', (err, user) => {
        if (err) {
            return res.status(403).json({ message: 'Invalid refresh token' });
        }

        // Check if refresh token exists in database
        const storedToken = db.getRefreshToken(user.id);
        if (storedToken !== refreshToken) {
            return res.status(403).json({ message: 'Refresh token not found' });
        }

        // Generate new access token
        const accessToken = generateAccessToken(user);
        res.json({ accessToken });
    });
});

#verify accces toekn route

const verifyAccessToken = (req, res, next) => {
    const token = req.headers['authorization'];

    if (!token) {
        return res.status(401).json({ message: 'Access token missing' });
    }

    jwt.verify(token, 'your-secret-key', (err, user) => {
        if (err) {
            return res.status(403).json({ message: 'Invalid access token' });
        }

        req.user = user;
        next();
    });
};

# here is the protected route

app.get('/profile', verifyAccessToken, (req, res) => {
    res.json({ message: 'This is your profile', user: req.user });
});
```

## ৫. ফ্রন্টএন্ডে ইমপ্লিমেন্টেশন

### টোকেনের মেয়াদ যাচাই করা:
```javascript
function isTokenExpired(token) {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const expiryTime = payload.exp * 1000;
    return Date.now() >= expiryTime;
}
```

### নতুন অ্যাক্সেস টোকেন পেতে API কল করা:
```javascript
async function refreshAccessToken() {
    try {
        const response = await fetch('/refresh-token', {
            method: 'POST',
            credentials: 'include',
        });

        if (response.ok) {
            const data = await response.json();
            return data.accessToken;
        } else {
            throw new Error('Failed to refresh token');
        }
    } catch (error) {
        console.error('Error refreshing token:', error);
        throw error;
    }
}
```

### অ্যাক্সেস টোকেনের মেয়াদ শেষ হলে নতুন টোকেন পেতে ফ্রন্টএন্ডে API কল:
```javascript
async function fetchWithAuth(url, options = {}) {
    let accessToken = localStorage.getItem('accessToken');

    if (isTokenExpired(accessToken)) {
        try {
            accessToken = await refreshAccessToken();
            localStorage.setItem('accessToken', accessToken);
        } catch (error) {
            window.location.href = '/login';
            return;
        }
    }

    options.headers = {
        ...options.headers,
        'Authorization': `Bearer ${accessToken}`,
    };

    const response = await fetch(url, options);
    return response;
}
```

### ফ্রন্টএন্ডে API কল করার উদাহরণ:
```javascript
async function fetchUserProfile() {
    try {
        const response = await fetchWithAuth('/api/profile');
        if (response.ok) {
            const data = await response.json();
            console.log('User Profile:', data);
        } else {
            console.error('Failed to fetch profile');
        }
    } catch (error) {
        console.error('Error:', error);
    }
}

fetchUserProfile();
```

# ফ্রন্টএন্ডে API কল করার উদাহরণ:

আপনি উপরের fetchWithAuth ফাংশনটি ব্যবহার করে যেকোনো প্রোটেক্টেড API কল করতে পারেন।

কোড উদাহরণ:
```javascript
async function fetchUserProfile() {
    try {
        const response = await fetchWithAuth('/api/profile');
        if (response.ok) {
            const data = await response.json();
            console.log('User Profile:', data);
        } else {
            console.error('Failed to fetch profile');
        }
    } catch (error) {
        console.error('Error:', error);
    }
}

fetchUserProfile();
```
