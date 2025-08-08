const express = require('express');
const cors = require('cors');
const { v4: uuidv4 } = require('uuid');
const path = require('path');
const fetch = require('node-fetch'); // <-- THIS IS THE MISSING MODULE

const app = express();
const PORT = process.env.PORT || 3000;

const scripts = {};
const keys = {};
const freeKeysGiven = new Set();
const plans = {
    '1-month': 30 * 24 * 60 * 60 * 1000,
    '5-months': 5 * 30 * 24 * 60 * 60 * 1000,
    '1-year': 365 * 24 * 60 * 60 * 1000,
    '2-years': 2 * 365 * 24 * 60 * 60 * 1000
};

app.use(cors());
app.use(express.json());

const kickScript = `
local message = "This link is already linked to another users account"
local player = game:GetService("Players").LocalPlayer
if player then
    player:Kick(message)
end
`;

app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'index.html'));
});

// --- API Endpoints for Frontend ---

app.get('/api/scripts', (req, res) => {
    res.json(Object.values(scripts));
});

app.post('/api/scripts', (req, res) => {
    const { code } = req.body;
    if (!code) {
        return res.status(400).json({ error: 'Code is required' });
    }
    const id = uuidv4();
    const key = uuidv4();
    scripts[id] = {
        id,
        key,
        code,
        whitelist: [],
        blacklist: [],
        executions: 0,
        firstExecutorId: null
    };
    console.log('New script created:', scripts[id]);
    res.status(201).json({ id, key });
});

app.delete('/api/scripts/:id', (req, res) => {
    const { id } = req.params;
    if (scripts[id]) {
        delete scripts[id];
        return res.status(204).send();
    }
    res.status(404).json({ error: 'Script not found' });
});

app.get('/api/users/:scriptId', (req, res) => {
    const { scriptId } = req.params;
    const script = scripts[scriptId];
    if (!script) {
        return res.status(404).json({ error: 'Script not found' });
    }
    res.json({
        whitelist: script.whitelist,
        blacklist: script.blacklist
    });
});

app.post('/api/users/:scriptId/:listType', (req, res) => {
    const { scriptId, listType } = req.params;
    const { userId } = req.body;
    const script = scripts[scriptId];

    if (!script) {
        return res.status(404).json({ error: 'Script not found' });
    }

    if (listType !== 'whitelist' && listType !== 'blacklist') {
        return res.status(400).json({ error: 'Invalid list type' });
    }

    if (!script[listType].includes(userId)) {
        script[listType].push(userId);
    }

    res.status(200).json(script[listType]);
});

app.delete('/api/users/:scriptId/:listType', (req, res) => {
    const { scriptId, listType } = req.params;
    const { userId } = req.body;
    const script = scripts[scriptId];

    if (!script) {
        return res.status(404).json({ error: 'Script not found' });
    }

    if (listType !== 'whitelist' && listType !== 'blacklist') {
        return res.status(400).json({ error: 'Invalid list type' });
    }

    const index = script[listType].indexOf(userId);
    if (index > -1) {
        script[listType].splice(index, 1);
    }

    res.status(200).json(script[listType]);
});

// --- NEW KEY GENERATION ENDPOINTS ---

app.post('/api/generate-key', async (req, res) => {
    const { plan } = req.body;
    if (!plans[plan]) {
        return res.status(400).json({ error: 'Invalid plan' });
    }

    const key = uuidv4();
    const expiresAt = Date.now() + plans[plan];
    keys[key] = { expiresAt, plan };
    console.log(`New ${plan} key generated: ${key}`);

    // Replace this URL with your actual Discord Webhook URL
    const webhookUrl = 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN';
    const embed = {
        title: "New Key Generated (Owner Panel)",
        color: 0x7b2cbf,
        fields: [
            {
                name: "Plan",
                value: plan,
                inline: true
            },
            {
                name: "Key",
                value: `\`\`\`${key}\`\`\``,
                inline: false
            },
            {
                name: "Expires At",
                value: new Date(expiresAt).toLocaleString(),
                inline: false
            }
        ],
        timestamp: new Date()
    };

    try {
        await fetch(webhookUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                embeds: [embed]
            })
        });
        res.status(200).json({ message: `Key generated and sent to Discord!`, plan, key });
    } catch (error) {
        console.error("Failed to send webhook message:", error);
        res.status(500).json({ error: 'Failed to generate key and send to webhook.' });
    }
});

// --- CORRECTED FREE KEY ENDPOINT ---
app.post('/api/free-key', async (req, res) => {
    const { userId } = req.body;
    if (!userId) {
        return res.status(400).json({ error: 'User ID is required.' });
    }

    if (freeKeysGiven.has(userId)) {
        return res.status(403).json({ error: 'You have already received your one-time free key.' });
    }

    const key = uuidv4();
    const expiresAt = Date.now() + plans['1-month'];
    keys[key] = { expiresAt, plan: '1-month', isFree: true };
    freeKeysGiven.add(userId);
    console.log(`New FREE 1-month key generated for user ${userId}: ${key}`);

    // Replace this URL with your actual Discord Webhook URL
    const webhookUrl = 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN';
    const embed = {
        title: "New FREE Key Generated",
        color: 0x2e8b57, // A green color for free keys
        fields: [
            {
                name: "User ID",
                value: userId,
                inline: true
            },
            {
                name: "Key",
                value: `\`\`\`${key}\`\`\``,
                inline: false
            },
            {
                name: "Expires At",
                value: new Date(expiresAt).toLocaleString(),
                inline: false
            }
        ],
        timestamp: new Date()
    };

    try {
        await fetch(webhookUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                embeds: [embed]
            })
        });
        res.status(200).json({ key, expiresAt, plan: '1-month' });
    } catch (error) {
        console.error("Failed to send webhook message:", error);
        res.status(500).json({ error: 'Failed to generate key and send to webhook.' });
    }
});

app.post('/api/check-key', (req, res) => {
    const { key } = req.body;
    const keyData = keys[key];

    if (!keyData) {
        return res.status(401).json({ error: 'Invalid key.' });
    }
    
    if (Date.now() > keyData.expiresAt) {
        delete keys[key];
        return res.status(401).json({ error: 'Key has expired.' });
    }

    res.status(200).json({ status: 'valid', plan: keyData.plan });
});

app.get('/raw/:id', (req, res) => {
    const { id } = req.params;
    const { key, userId } = req.query;
    const script = scripts[id];

    if (!scripts[id] || scripts[id].key !== key) {
        console.log(`Unauthorized access attempt for script ${id} with key ${key}`);
        return res.status(403).send('Unauthorized');
    }

    if (script.firstExecutorId) {
        if (script.firstExecutorId !== userId) {
            console.log(`Unauthorized execution attempt by user ${userId} for script ${id}`);
            return res.type('text/plain').send(kickScript);
        }
    } else {
        script.firstExecutorId = userId;
        console.log(`Script ${id} is now bound to user ${userId}`);
    }

    if (script.blacklist.includes(userId)) {
        console.log(`Blacklisted user ${userId} attempted to run script ${id}`);
        return res.type('text/plain').send(`
            local player = game:GetService("Players").LocalPlayer
            if player then
                player:Kick("You are on the blacklist for this script.")
            end
        `);
    }

    if (script.whitelist.length > 0 && !script.whitelist.includes(userId)) {
        console.log(`User ${userId} not on whitelist for script ${id}`);
        return res.type('text/plain').send(`
            local player = game:GetService("Players").LocalPlayer
            if player then
                player:Kick("You are not on the whitelist for this script.")
            end
        `);
    }

    script.executions = (script.executions || 0) + 1;
    console.log(`Script ${id} executed by user ${userId}. Total executions: ${script.executions}`);

    res.type('text/plain').send(script.code);
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
