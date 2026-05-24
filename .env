// ============================================
// CLINIC APPOINTMENT SCHEDULER - BACKEND CODE
// Tech: Node.js + Express + MongoDB
// ============================================

// ===== 1. PROJECT SETUP =====
// Run these commands in terminal:
/*
mkdir clinic-scheduler-backend
cd clinic-scheduler-backend
npm init -y
npm install express mongodb dotenv axios twilio node-cron cors
npm install -D nodemon

Create .env file with:
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/clinic-scheduler
TWILIO_SID=your_twilio_sid
TWILIO_AUTH_TOKEN=your_twilio_token
TWILIO_PHONE=whatsapp:+14155552671
CLAUDE_API_KEY=your_claude_key
PORT=3000
JWT_SECRET=your_secret_key
*/

// ===== 2. SERVER SETUP =====
// File: server.js

const express = require('express');
const { MongoClient } = require('mongodb');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

let db;

// Connect to MongoDB
MongoClient.connect(process.env.MONGODB_URI, { useUnifiedTopology: true })
  .then(client => {
    console.log('Connected to MongoDB');
    db = client.db('clinic-scheduler');
  })
  .catch(error => console.error(error));

// ===== ROUTES =====

// 1. CLINIC ROUTES
app.post('/api/clinics', async (req, res) => {
  try {
    const clinic = {
      name: req.body.name,
      email: req.body.email,
      phone: req.body.phone,
      
      // AI-driven profile
      aiProfile: {
        specialty: req.body.specialty || "general",
        appointmentDuration: req.body.appointmentDuration || 30,
        followUpDays: req.body.followUpDays || [7, 30],
        messageTone: req.body.messageTone || "professional",
        language: req.body.language || "en",
        maxRemindersSent: req.body.maxRemindersSent || 3,
        reminderTime: req.body.reminderTime || "10:00 AM"
      },
      
      workingHours: req.body.workingHours || {
        monday: { start: "09:00", end: "18:00" },
        tuesday: { start: "09:00", end: "18:00" },
        wednesday: { start: "09:00", end: "18:00" },
        thursday: { start: "09:00", end: "18:00" },
        friday: { start: "09:00", end: "18:00" },
        saturday: { start: "10:00", end: "14:00" },
        sunday: null
      },
      
      messageTemplates: req.body.messageTemplates || {
        appointment_confirmed: "Your appointment is confirmed for {{date}} at {{time}}. Reply YES to confirm.",
        reminder_24h: "Reminder: Your appointment is tomorrow at {{time}}.",
        followup: "How was your visit? We'd love your feedback!"
      },
      
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    const result = await db.collection('clinics').insertOne(clinic);
    res.status(201).json({ 
      message: "Clinic registered successfully",
      clinicId: result.insertedId 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get clinic profile
app.get('/api/clinics/:id', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const clinic = await db.collection('clinics').findOne({
      _id: new ObjectId(req.params.id)
    });
    
    if (!clinic) {
      return res.status(404).json({ error: "Clinic not found" });
    }
    
    res.json(clinic);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update clinic AI profile
app.put('/api/clinics/:id/profile', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const result = await db.collection('clinics').updateOne(
      { _id: new ObjectId(req.params.id) },
      { 
        $set: {
          aiProfile: req.body.aiProfile,
          updatedAt: new Date()
        }
      }
    );
    
    res.json({ message: "Profile updated", modifiedCount: result.modifiedCount });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ===== 2. PATIENT ROUTES =====

app.post('/api/patients', async (req, res) => {
  try {
    const patient = {
      name: req.body.name,
      phone: req.body.phone, // WhatsApp number
      email: req.body.email || null,
      clinicId: req.body.clinicId,
      
      medicalHistory: req.body.medicalHistory || "",
      allergies: req.body.allergies || "",
      
      // Track reminders sent
      remindersSent: 0,
      lastAppointment: null,
      
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    const result = await db.collection('patients').insertOne(patient);
    res.status(201).json({ 
      message: "Patient added",
      patientId: result.insertedId 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ===== 3. APPOINTMENT ROUTES =====

app.post('/api/appointments', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    
    const appointment = {
      clinicId: new ObjectId(req.body.clinicId),
      patientId: new ObjectId(req.body.patientId),
      patientName: req.body.patientName,
      patientPhone: req.body.patientPhone,
      
      appointmentDate: new Date(req.body.appointmentDate),
      appointmentTime: req.body.appointmentTime, // "2:30 PM"
      doctorName: req.body.doctorName || "Doctor",
      reason: req.body.reason || "",
      
      status: "scheduled", // scheduled, confirmed, completed, cancelled
      confirmationStatus: "pending", // pending, confirmed, unconfirmed
      reminderSent: false,
      noShowStatus: null, // null, no_show, cancelled
      
      followUpScheduled: false,
      
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    const result = await db.collection('appointments').insertOne(appointment);
    res.status(201).json({ 
      message: "Appointment scheduled",
      appointmentId: result.insertedId 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get clinic's appointments
app.get('/api/clinics/:id/appointments', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const appointments = await db.collection('appointments')
      .find({ clinicId: new ObjectId(req.params.id) })
      .sort({ appointmentDate: 1 })
      .toArray();
    
    res.json(appointments);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get today's appointments
app.get('/api/clinics/:id/appointments/today', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const tomorrow = new Date(today);
    tomorrow.setDate(tomorrow.getDate() + 1);
    
    const appointments = await db.collection('appointments')
      .find({
        clinicId: new ObjectId(req.params.id),
        appointmentDate: {
          $gte: today,
          $lt: tomorrow
        }
      })
      .sort({ appointmentTime: 1 })
      .toArray();
    
    res.json(appointments);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update appointment status
app.put('/api/appointments/:id', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const result = await db.collection('appointments').updateOne(
      { _id: new ObjectId(req.params.id) },
      { 
        $set: {
          status: req.body.status,
          confirmationStatus: req.body.confirmationStatus,
          noShowStatus: req.body.noShowStatus,
          updatedAt: new Date()
        }
      }
    );
    
    res.json({ message: "Appointment updated", modifiedCount: result.modifiedCount });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ===== 4. WHATSAPP MESSAGING =====

const twilio = require('twilio');

async function sendWhatsAppMessage(toPhone, message) {
  try {
    const client = twilio(process.env.TWILIO_SID, process.env.TWILIO_AUTH_TOKEN);
    
    const response = await client.messages.create({
      from: process.env.TWILIO_PHONE, // e.g., 'whatsapp:+14155552671'
      to: `whatsapp:${toPhone}`,
      body: message
    });
    
    console.log(`Message sent: ${response.sid}`);
    return response.sid;
  } catch (error) {
    console.error(`Failed to send message: ${error}`);
    throw error;
  }
}

// Manual message send (from dashboard)
app.post('/api/messages/send', async (req, res) => {
  try {
    const { appointmentId, message } = req.body;
    const { ObjectId } = require('mongodb');
    
    const appointment = await db.collection('appointments').findOne({
      _id: new ObjectId(appointmentId)
    });
    
    if (!appointment) {
      return res.status(404).json({ error: "Appointment not found" });
    }
    
    const messageSid = await sendWhatsAppMessage(
      appointment.patientPhone,
      message
    );
    
    // Log the message
    await db.collection('messages').insertOne({
      appointmentId: new ObjectId(appointmentId),
      patientPhone: appointment.patientPhone,
      message: message,
      type: "manual",
      messageSid: messageSid,
      sentAt: new Date(),
      status: "sent"
    });
    
    res.json({ 
      message: "Message sent successfully",
      messageSid: messageSid
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Receive WhatsApp messages (webhook)
app.post('/api/whatsapp/webhook', express.urlencoded({ extended: false }), async (req, res) => {
  try {
    const from = req.body.From; // whatsapp:+919876543210
    const messageBody = req.body.Body.toLowerCase();
    
    // Extract phone number
    const patientPhone = from.replace('whatsapp:', '');
    
    // Find appointment
    const appointment = await db.collection('appointments').findOne({
      patientPhone: patientPhone
    });
    
    if (!appointment) {
      res.send('<Response></Response>');
      return;
    }
    
    // Handle confirmation
    if (messageBody.includes('yes') || messageBody.includes('confirm')) {
      await db.collection('appointments').updateOne(
        { _id: appointment._id },
        { $set: { confirmationStatus: 'confirmed' } }
      );
      
      await sendWhatsAppMessage(
        patientPhone,
        "Perfect! Your appointment is confirmed. See you soon!"
      );
    }
    
    // Handle cancellation
    else if (messageBody.includes('cancel') || messageBody.includes('no')) {
      await db.collection('appointments').updateOne(
        { _id: appointment._id },
        { $set: { status: 'cancelled', noShowStatus: 'cancelled' } }
      );
      
      await sendWhatsAppMessage(
        patientPhone,
        "Your appointment has been cancelled. Feel free to reschedule anytime!"
      );
    }
    
    res.send('<Response></Response>');
  } catch (error) {
    console.error(error);
    res.send('<Response></Response>');
  }
});

// ===== 5. AI MESSAGE GENERATION =====

const Anthropic = require('@anthropic-ai/sdk');

async function generateAIMessage(appointmentData, clinic, messageType) {
  try {
    const client = new Anthropic({
      apiKey: process.env.CLAUDE_API_KEY
    });
    
    const prompts = {
      reminder_24h: `Generate a WhatsApp reminder message for a patient appointment.
        
        Clinic: ${clinic.name} (${clinic.aiProfile.specialty})
        Patient: ${appointmentData.patientName}
        Date: ${appointmentData.appointmentDate}
        Time: ${appointmentData.appointmentTime}
        Doctor: ${appointmentData.doctorName}
        
        Tone: ${clinic.aiProfile.messageTone}
        Language: ${clinic.aiProfile.language === 'hi' ? 'Hindi (with English mix)' : 'English'}
        
        Keep it brief (1-2 sentences), friendly and professional.
        Ask them to reply "YES" to confirm or "CANCEL" to reschedule.`,
      
      followup: `Generate a follow-up message after a patient's visit.
        
        Clinic: ${clinic.name}
        Patient: ${appointmentData.patientName}
        Visit Reason: ${appointmentData.reason}
        
        Tone: ${clinic.aiProfile.messageTone}
        Language: ${clinic.aiProfile.language === 'hi' ? 'Hindi' : 'English'}
        
        Ask how their visit went and if they need any follow-up. Keep it to 1 sentence.`,
      
      confirmation: `Generate an appointment confirmation message.
        
        Clinic: ${clinic.name}
        Patient: ${appointmentData.patientName}
        Date: ${appointmentData.appointmentDate}
        Time: ${appointmentData.appointmentTime}
        
        Tone: ${clinic.aiProfile.messageTone}
        
        Make it warm and professional. 1 sentence.`
    };
    
    const response = await client.messages.create({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 100,
      messages: [
        {
          role: "user",
          content: prompts[messageType] || prompts.reminder_24h
        }
      ]
    });
    
    return response.content[0].text;
  } catch (error) {
    console.error('AI message generation failed:', error);
    // Fallback message
    return `Hi ${appointmentData.patientName}, your appointment is at ${appointmentData.appointmentTime}. Please reply YES to confirm.`;
  }
}

// Send AI-generated reminder
app.post('/api/reminders/send-ai', async (req, res) => {
  try {
    const { appointmentId } = req.body;
    const { ObjectId } = require('mongodb');
    
    const appointment = await db.collection('appointments').findOne({
      _id: new ObjectId(appointmentId)
    });
    
    const clinic = await db.collection('clinics').findOne({
      _id: appointment.clinicId
    });
    
    // Generate AI message
    const aiMessage = await generateAIMessage(appointment, clinic, 'reminder_24h');
    
    // Send via WhatsApp
    const messageSid = await sendWhatsAppMessage(appointment.patientPhone, aiMessage);
    
    // Mark as sent
    await db.collection('appointments').updateOne(
      { _id: new ObjectId(appointmentId) },
      { $set: { reminderSent: true } }
    );
    
    res.json({ 
      message: "AI reminder sent",
      generatedMessage: aiMessage,
      messageSid: messageSid
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ===== 6. SCHEDULED REMINDERS (CRON) =====

const cron = require('node-cron');

// Run every hour to send reminders for next-day appointments
cron.schedule('0 * * * *', async () => {
  try {
    const tomorrow = new Date();
    tomorrow.setDate(tomorrow.getDate() + 1);
    tomorrow.setHours(0, 0, 0, 0);
    
    const tomorrowEnd = new Date(tomorrow);
    tomorrowEnd.setHours(23, 59, 59, 999);
    
    const appointments = await db.collection('appointments').find({
      appointmentDate: {
        $gte: tomorrow,
        $lt: tomorrowEnd
      },
      reminderSent: false,
      status: "scheduled"
    }).toArray();
    
    for (const appt of appointments) {
      const clinic = await db.collection('clinics').findOne({
        _id: appt.clinicId
      });
      
      const aiMessage = await generateAIMessage(appt, clinic, 'reminder_24h');
      await sendWhatsAppMessage(appt.patientPhone, aiMessage);
      
      await db.collection('appointments').updateOne(
        { _id: appt._id },
        { $set: { reminderSent: true } }
      );
      
      console.log(`Reminder sent to ${appt.patientName}`);
    }
  } catch (error) {
    console.error('Reminder scheduling error:', error);
  }
});

// ===== 7. ANALYTICS =====

app.get('/api/clinics/:id/analytics', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const clinicId = new ObjectId(req.params.id);
    
    const appointments = await db.collection('appointments')
      .find({ clinicId })
      .toArray();
    
    const total = appointments.length;
    const confirmed = appointments.filter(a => a.confirmationStatus === 'confirmed').length;
    const noShows = appointments.filter(a => a.noShowStatus === 'no_show').length;
    const completed = appointments.filter(a => a.status === 'completed').length;
    
    const confirmationRate = total > 0 ? ((confirmed / total) * 100).toFixed(2) : 0;
    const noShowRate = total > 0 ? ((noShows / total) * 100).toFixed(2) : 0;
    
    res.json({
      totalAppointments: total,
      confirmed: confirmed,
      confirmationRate: `${confirmationRate}%`,
      noShows: noShows,
      noShowRate: `${noShowRate}%`,
      completed: completed
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ===== START SERVER =====

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Clinic Scheduler API running on http://localhost:${PORT}`);
});
