import express, { Request, Response } from 'express';
import path from 'path';
import { createServer as createViteServer } from 'vite';
import dotenv from 'dotenv';
import { GoogleGenAI } from '@google/genai';
import {
  NUTRITION_ITEMS,
  BODYBUILDER_SPLITS,
  WORKOUTS_DATA,
  MUSCLE_RECOVERY_TIMELINE,
  FAQ_LIST
} from './src/data/fitnessData.ts';

dotenv.config();

const app = express();
const PORT = 3000;

app.use(express.json());

// In-memory submissions store
const contactInquiries: Array<{
  id: string;
  name: string;
  email: string;
  topic: string;
  message: string;
  createdAt: string;
  coachReply: string;
}> = [];

// Lazy initialized Gemini client
let genAI: GoogleGenAI | null = null;
function getGeminiClient(): GoogleGenAI | null {
  if (!genAI && process.env.GEMINI_API_KEY) {
    try {
      genAI = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
    } catch (err) {
      console.error('Failed to initialize Gemini client:', err);
    }
  }
  return genAI;
}

// 1. Health check
app.get('/api/health', (req: Request, res: Response) => {
  res.json({
    status: 'healthy',
    system: 'IronVault Fitness Backend',
    timestamp: new Date().toISOString()
  });
});

// 2. Comprehensive fitness data endpoint
app.get('/api/fitness/data', (req: Request, res: Response) => {
  res.json({
    nutrition: NUTRITION_ITEMS,
    splits: BODYBUILDER_SPLITS,
    workouts: WORKOUTS_DATA,
    recovery: MUSCLE_RECOVERY_TIMELINE,
    faqs: FAQ_LIST
  });
});

// 3. Contact & Help desk submission endpoint
app.post('/api/contact', async (req: Request, res: Response) => {
  try {
    const { name, email, topic, message } = req.body;

    if (!name || !email || !message) {
      return res.status(400).json({
        success: false,
        error: 'Name, email, and message are required fields.'
      });
    }

    // Auto-generate expert coach guidance based on topic
    let coachReply = '';
    const lowerTopic = (topic || '').toLowerCase();

    if (lowerTopic.includes('nutrition')) {
      coachReply = `Coach Note for ${name}: For your nutrition goal, prioritize 2.0g protein/kg bodyweight. Focus on whole foods: chicken breast, salmon, oats, sweet potatoes, and tart cherry post-workout.`;
    } else if (lowerTopic.includes('split')) {
      coachReply = `Coach Note for ${name}: Consistency beats novelty. For bulking, stick to a 4-6 day high volume progressive overload split (like Arnold's or Ronnie's) for 12 weeks minimum. For cutting, switch to high-intensity density with a 500 kcal deficit.`;
    } else if (lowerTopic.includes('workout')) {
      coachReply = `Coach Note for ${name}: Prioritize strict eccentric control (2-3 second descent) over ego lifting. Ensure your form matches the movement cues to fully recruit target fibers.`;
    } else if (lowerTopic.includes('recovery')) {
      coachReply = `Coach Note for ${name}: Remember muscles grow during rest, not during training. Ensure 7.5-9 hours of sleep, hydrate with 3.5L+ electrolytes, and respect the 48-72h recovery window before hitting the same muscle again.`;
    } else {
      coachReply = `Coach Note for ${name}: Your inquiry has been logged in the IronVault Help Desk. Stay disciplined, track your lifts, and keep pushing limits!`;
    }

    const coachEmail = 'g.sanathkumar2008@gmail.com';

    const newInquiry = {
      id: `inq-${Date.now()}`,
      name: String(name).trim(),
      email: String(email).trim(),
      topic: String(topic || 'general'),
      message: String(message).trim(),
      recipientCoachEmail: coachEmail,
      createdAt: new Date().toISOString(),
      coachReply
    };

    contactInquiries.unshift(newInquiry);

    const gmailWebComposeUrl = `https://mail.google.com/mail/?view=cm&fs=1&to=${encodeURIComponent(coachEmail)}&su=${encodeURIComponent(`[IronVault Fitness] Inquiry from ${name}: ${topic}`)}&body=${encodeURIComponent(`Athlete Name: ${name}\nContact Email: ${email}\nTopic: ${topic}\n\nInquiry Details:\n${message}\n\n---\nTransmitted via IronVault Fitness Coaching Desk`)}`;
    const mailtoUrl = `mailto:${coachEmail}?subject=${encodeURIComponent(`[IronVault Fitness] Inquiry from ${name}: ${topic}`)}&body=${encodeURIComponent(`Athlete Name: ${name}\nContact Email: ${email}\nTopic: ${topic}\n\nInquiry Details:\n${message}`)}`;

    return res.status(200).json({
      success: true,
      message: `Inquiry successfully logged for Coach Sanath Kumar (${coachEmail}).`,
      coachEmail,
      gmailWebComposeUrl,
      mailtoUrl,
      inquiry: newInquiry
    });
  } catch (err: any) {
    console.error('Error processing contact inquiry:', err);
    return res.status(500).json({
      success: false,
      error: 'Internal server error while processing your inquiry.'
    });
  }
});

// 4. Interactive Gym Coach AI endpoint (Gemini or expert gym engine)
app.post('/api/ai/coach', async (req: Request, res: Response) => {
  try {
    const { prompt, goal, currentWeight, targetWeight, trainingLevel } = req.body;

    if (!prompt) {
      return res.status(400).json({ error: 'Prompt is required.' });
    }

    const ai = getGeminiClient();

    if (ai && process.env.GEMINI_API_KEY) {
      try {
        const systemInstruction = `You are "IronVault Elite Coach", a world-class bodybuilding and fitness coach inspired by Golden Era and modern bodybuilding science.
Provide concise, high-impact, direct, and actionable advice. Focus on:
1. Exact exercise mechanics & progressive overload
2. Caloric guidelines & macro targets (Protein, Carbs, Fats)
3. Muscle recovery timelines & DOMS management
Keep responses structured with bold bullet points, motivating tone, and no fluff.`;

        const userContext = `User Question: "${prompt}"
Context: Goal=${goal || 'General Fitness'}, Weight=${currentWeight || 'N/A'}, Target=${targetWeight || 'N/A'}, Level=${trainingLevel || 'Intermediate'}`;

        const response = await ai.models.generateContent({
          model: 'gemini-2.5-flash',
          contents: `${systemInstruction}\n\n${userContext}`
        });

        const reply = response.text || 'Keep your intensity high and maintain strict form!';
        return res.json({ advice: reply, source: 'gemini-ai' });
      } catch (geminiError) {
        console.warn('Gemini API call failed, using expert gym fallback:', geminiError);
      }
    }

    // Expert gym rule-based fallback
    let fallbackAdvice = `**IronVault Pro Coach Advice:**\n\n`;
    const lowerPrompt = prompt.toLowerCase();

    if (lowerPrompt.includes('bulk') || lowerPrompt.includes('gain mass')) {
      fallbackAdvice += `• **Calorie Surplus**: Aim for +350 to +500 kcal above maintenance to maximize muscle protein synthesis with minimal fat gain.\n• **Split Recommendation**: Adopt the 6-Day Arnold Golden Era or Ronnie Coleman Heavy Mass PPL.\n• **Fuel Staple**: Clean carbohydrates (jasmine rice, oats, sweet potatoes) combined with 2.2g protein per kg.\n• **Recovery**: 48-72h between heavy muscle training with 8 hours of deep restorative sleep.`;
    } else if (lowerPrompt.includes('cut') || lowerPrompt.includes('lose fat') || lowerPrompt.includes('shred')) {
      fallbackAdvice += `• **Calorie Deficit**: Target a -400 to -500 kcal deficit. Preserve muscle strength by keeping heavy compound sets.\n• **Split Recommendation**: CBum Classic Conditioning 5-Day split with 30 mins fasted morning incline walking.\n• **Protein Shield**: Increase protein to 2.4g/kg to prevent catabolism during calorie restriction.\n• **Hydration**: Drink 4+ liters of water with Himalayan pink salt to maintain cell fullness.`;
    } else if (lowerPrompt.includes('chest') || lowerPrompt.includes('bench')) {
      fallbackAdvice += `• **Upper Chest Priority**: Incline Barbell or Dumbbell Press at 30 degrees (4 sets x 8-10 reps).\n• **Form Cue**: Retract and depress scapula; lower bar smoothly without bouncing off sternum.\n• **Recovery Window**: Chest pectorals need 48-72 hours to rebuild micro-tears.`;
    } else if (lowerPrompt.includes('leg') || lowerPrompt.includes('squat')) {
      fallbackAdvice += `• **Foundation**: Barbell back squats below parallel (4 sets x 6-10 reps) followed by Romanian Deadlifts.\n• **Quad Sweep**: 45-degree leg press with high foot placement for glutes or low placement for quads.\n• **Recovery Window**: Legs are your largest muscle group and demand 72-96 hours of recovery before another brutal session!`;
    } else {
      fallbackAdvice += `• **Progressive Overload**: Strive to add 1 rep or 1-2.5 kg to your main compound lifts each week.\n• **Nutrition**: Fuel your workouts with pre-workout carbs (oats/fruit) and fast-absorbing whey isolate post-workout.\n• **Muscle Repair**: Respect muscle recovery time (48-72 hours) and fuel with anti-inflammatory foods like wild salmon and tart cherry juice.`;
    }

    return res.json({ advice: fallbackAdvice, source: 'ironvault-coach' });
  } catch (error: any) {
    console.error('AI Coach Error:', error);
    return res.status(500).json({ error: 'Failed to generate coach advice.' });
  }
});

// Production and Development Vite Middleware
async function startServer() {
  if (process.env.NODE_ENV !== 'production') {
    const vite = await createViteServer({
      server: { middlewareMode: true },
      appType: 'spa',
    });
    app.use(vite.middlewares);
  } else {
    const distPath = path.join(process.cwd(), 'dist');
    app.use(express.static(distPath));
    app.get('*', (req: Request, res: Response) => {
      res.sendFile(path.join(distPath, 'index.html'));
    });
  }

  app.listen(PORT, '0.0.0.0', () => {
    console.log(`IronVault Fitness Server running on http://0.0.0.0:${PORT}`);
  });
}

startServer();
