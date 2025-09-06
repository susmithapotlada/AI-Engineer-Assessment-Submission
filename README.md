import React, { useMemo, useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Badge } from "@/components/ui/badge";
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
import { Select, SelectTrigger, SelectContent, SelectItem, SelectValue } from "@/components/ui/select";
import { Label } from "@/components/ui/label";
import { Search, Send, Mail, Clock, AlertTriangle, Inbox, Filter, RefreshCw } from "lucide-react";
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, PieChart, Pie, Cell } from "recharts";
import { motion } from "framer-motion";

// ---------- Mock Data (replace with API calls) ----------
const MOCK_EMAILS = [
  {
    id: "em_001",
    from: "anuj@contoso.com",
    subject: "Cannot access dashboard — URGENT HELP",
    body: "Hi team, I cannot access the analytics dashboard since last night. It says '403 forbidden'. This is critical for my client demo today at 4 PM. Please fix immediately. My phone: +91-90000-12345",
    receivedAt: "2025-09-06T07:40:00Z",
    product: "Contoso Analytics",
  },
  {
    id: "em_002",
    from: "meera@acmecorp.io",
    subject: "Support: Invoice query for August",
    body: "Hello, could you clarify why my August invoice shows two pro subscriptions? Alternate email: billing@acmecorp.io",
    receivedAt: "2025-09-06T05:10:00Z",
    product: "Pro Subscription",
  },
  {
    id: "em_003",
    from: "rahul@startify.app",
    subject: "Request: Feature toggle for beta",
    body: "We love the product! Could you enable the new beta toggle for org id 7781 for next week's launch?",
    receivedAt: "2025-09-05T16:55:00Z",
    product: "Startify Cloud",
  },
  {
    id: "em_004",
    from: "priya@globex.com",
    subject: "Query: Password reset emails not received",
    body: "Hi Support, our users report that password reset emails are not arriving. This is impacting onboarding today.",
    receivedAt: "2025-09-06T06:30:00Z",
    product: "Auth Service",
  },
];

// ---------- Lightweight NLP Utilities (demo only) ----------
const SENTIMENT_LEXICON = {
  positive: ["love", "thanks", "great", "awesome", "appreciate"],
  negative: ["cannot", "error", "forbidden", "not", "issue", "urgent", "critical", "immediately"],
};

const PRIORITY_KEYWORDS = ["urgent", "immediately", "critical", "cannot access", "down", "priority", "asap", "impacting"];

function detectSentiment(text) {
  const t = text.toLowerCase();
  const pos = SENTIMENT_LEXICON.positive.some((w) => t.includes(w));
  const neg = SENTIMENT_LEXICON.negative.some((w) => t.includes(w));
  if (neg && !pos) return "Negative";
  if (pos && !neg) return "Positive";
  return "Neutral";
}

function detectPriority(email) {
  const t = (email.subject + " " + email.body).toLowerCase();
  const urgent = PRIORITY_KEYWORDS.some((w) => t.includes(w));
  return urgent ? "Urgent" : "Not urgent";
}

function extractInfo(text) {
  const phone = (text.match(/\+?\d[\d\s-]{7,}/g) || []).map((s) => s.trim());
  const emails = (text.match(/[\w.-]+@[\w.-]+\.[A-Za-z]{2,}/g) || []).map((s) => s.trim());
  // Naive intent extraction: look for nouns/verbs by keywords
  const intents = [];
  if (/invoice|billing/i.test(text)) intents.push("Billing");
  if (/access|login|403|password/i.test(text)) intents.push("Access/Authentication");
  if (/feature|enable|beta|toggle/i.test(text)) intents.push("Feature Request");
  if (/reset/i.test(text)) intents.push("Reset");
  return { phone, emails, intents };
}

function draftReply(email, sentiment, priority, info) {
  const greeting = `Hi ${email.from.split("@")[0][0].toUpperCase()}${email.from.split("@")[0].slice(1)},`;
  const empathy = sentiment === "Negative" ?
    "I'm sorry for the trouble—thanks for flagging this promptly." :
    "Thanks for reaching out!";
  let body = "";
  if (info.intents.includes("Access/Authentication")) {
    body = `We've seen your note about access to ${email.product}. Our team is investigating right now. If possible, please confirm the affected user/email and approximate time of failure. We'll update you within 30–45 minutes.`;
  } else if (info.intents.includes("Billing")) {
    body = `We can help clarify your invoice for ${email.product}. Could you share the invoice number and PO (if any)? In the meantime, we are checking for duplicate subscriptions on your account.`;
  } else if (info.intents.includes("Feature Request")) {
    body = `We've queued the beta feature toggle request (org 7781 noted if applicable). We'll confirm enablement and rollout windows shortly.`;
  } else {
    body = `We've received your request and are looking into it. We'll circle back with an update shortly.`;
  }
  const sig = `\n\nBest regards,\nSupport Assistant`;
  const priorityLine = priority === "Urgent" ? "\n\nWe recognize this as urgent and have prioritized it." : "";
  return `${greeting}\n\n${empathy} ${body}${priorityLine}${sig}`;
}

const prepareEmail = (e) => {
  const sentiment = detectSentiment(e.subject + " " + e.body);
  const priority = detectPriority(e);
  const info = extractInfo(e.body);
  const draft = draftReply(e, sentiment, priority, info);
  return { ...e, sentiment, priority, info, draft };
};

// ---------- UI Components ----------
function Stat({ label, value, icon: Icon }) {
  return (
    <Card className="rounded-2xl shadow-sm">
      <CardContent className="p-4 flex items-center gap-3">
        <div className="p-2 rounded-xl bg-muted"><Icon className="w-5 h-5" /></div>
        <div>
          <div className="text-xs text-muted-foreground">{label}</div>
          <div className="text-2xl font-semibold">{value}</div>
        </div>
      </CardContent>
    </Card>
  );
}

function EmailRow({ item, onSelect }) {
  return (
    <motion.div layout initial={{ opacity: 0, y: 6 }} animate={{ opacity: 1, y: 0 }}>
      <button onClick={() => onSelect(item)} className="w-full text-left">
        <Card className="rounded-2xl hover:shadow-md transition">
          <CardContent className="p-4 grid grid-cols-12 items-start gap-3">
            <div className="col-span-12 md:col-span-7">
              <div className="flex items-center gap-2">
                <Badge variant="secondary" className="rounded-full">{item.priority}</Badge>
                <Badge className="rounded-full" variant={item.sentiment === "Negative" ? "destructive" : item.sentiment === "Positive" ? "default" : "secondary"}>{item.sentiment}</Badge>
                <Badge className="rounded-full" variant="outline">{new Date(item.receivedAt).toLocaleString()}</Badge>
              </div>
              <div className="mt-2 font-medium line-clamp-1">{item.subject}</div>
              <div className="text-sm text-muted-foreground line-clamp-2">{item.body}</div>
            </div>
            <div className="col-span-12 md:col-span-3 text-sm">
              <div className="font-medium"><Mail className="inline w-4 h-4 mr-1" />{item.from}</div>
              {item.info.intents.length > 0 && (
                <div className="mt-2 flex flex-wrap gap-2">
                  {item.info.intents.map((t) => (
                    <Badge key={t} variant="outline" className="rounded-full">{t}</Badge>
                  ))}
                </div>
              )}
            </div>
            <div className="col-span-12 md:col-span-2 flex justify-end">
              <Button size="sm" variant="outline">Review</Button>
            </div>
          </CardContent>
        </Card>
      </button>
    </motion.div>
  );
}

export default function App() {
  const [query, setQuery] = useState("");
  const enriched = useMemo(() => MOCK_EMAILS.map(prepareEmail), []);
  const [data, setData] = useState(enriched);
  const [selected, setSelected] = useState(data[0]);
  const [priorityFilter, setPriorityFilter] = useState("All");
  const [sentimentFilter, setSentimentFilter] = useState("All");

  const filtered = useMemo(() => {
    return data
      .filter((e) =>
        [e.subject, e.body, e.from, e.product].some((s) => s.toLowerCase().includes(query.toLowerCase()))
      )
      .filter((e) => (priorityFilter === "All" ? true : e.priority === priorityFilter))
      .filter((e) => (sentimentFilter === "All" ? true : e.sentiment === sentimentFilter))
      .sort((a, b) => (a.priority === "Urgent" && b.priority !== "Urgent" ? -1 : 1));
  }, [data, query, priorityFilter, sentimentFilter]);

  const stats = useMemo(() => {
    const total = data.length;
    const urgent = data.filter((x) => x.priority === "Urgent").length;
    const resolved = 1; // mock
    const pending = total - resolved;
    return { total, urgent, resolved, pending };
  }, [data]);

  const categoryChart = useMemo(() => {
    const sentiments = ["Positive", "Neutral", "Negative"];
    return sentiments.map((s) => ({ name: s, value: data.filter((e) => e.sentiment === s).length }));
  }, [data]);

  const priorityChart = useMemo(() => {
    return [
      { name: "Urgent", value: data.filter((e) => e.priority === "Urgent").length },
      { name: "Not urgent", value: data.filter((e) => e.priority === "Not urgent").length },
    ];
  }, [data]);

  return (
    <div className="min-h-screen bg-background">
      <header className="sticky top-0 z-10 backdrop-blur supports-[backdrop-filter]:bg-background/60 border-b">
        <div className="max-w-7xl mx-auto p-4 flex items-center gap-3">
          <Inbox className="w-6 h-6" />
          <h1 className="text-xl font-semibold">AI-Powered Support Inbox</h1>
          <div className="ml-auto flex items-center gap-2">
            <Button variant="outline" size="sm"><RefreshCw className="w-4 h-4 mr-1" />Sync</Button>
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto p-4 grid grid-cols-12 gap-4">
        {/* Left: List & Filters */}
        <div className="col-span-12 lg:col-span-7 space-y-4">
          <Card className="rounded-2xl">
            <CardHeader className="pb-2">
              <CardTitle className="text-base">Filters</CardTitle>
            </CardHeader>
            <CardContent className="grid md:grid-cols-4 gap-3">
              <div className="md:col-span-2">
                <div className="relative">
                  <Search className="absolute left-2 top-2.5 w-4 h-4 text-muted-foreground" />
                  <Input className="pl-8" placeholder="Search subject, body, sender" value={query} onChange={(e) => setQuery(e.target.value)} />
                </div>
              </div>
              <div>
                <Label className="text-xs">Priority</Label>
                <Select value={priorityFilter} onValueChange={setPriorityFilter}>
                  <SelectTrigger><SelectValue placeholder="All" /></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="All">All</SelectItem>
                    <SelectItem value="Urgent">Urgent</SelectItem>
                    <SelectItem value="Not urgent">Not urgent</SelectItem>
                  </SelectContent>
                </Select>
              </div>
              <div>
                <Label className="text-xs">Sentiment</Label>
                <Select value={sentimentFilter} onValueChange={setSentimentFilter}>
                  <SelectTrigger><SelectValue placeholder="All" /></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="All">All</SelectItem>
                    <SelectItem value="Positive">Positive</SelectItem>
                    <SelectItem value="Neutral">Neutral</SelectItem>
                    <SelectItem value="Negative">Negative</SelectItem>
                  </SelectContent>
                </Select>
              </div>
            </CardContent>
          </Card>

          <div className="space-y-3">
            {filtered.map((item) => (
              <EmailRow key={item.id} item={item} onSelect={setSelected} />
            ))}
          </div>
        </div>

        {/* Right: Details & Draft */}
        <div className="col-span-12 lg:col-span-5 space-y-4">
          <div className="grid grid-cols-2 gap-3">
            <Stat label="Total (24h)" value={stats.total} icon={Inbox} />
            <Stat label="Urgent" value={stats.urgent} icon={AlertTriangle} />
            <Stat label="Resolved" value={stats.resolved} icon={Send} />
            <Stat label="Pending" value={stats.pending} icon={Clock} />
          </div>

          <Card className="rounded-2xl">
            <CardHeader>
              <CardTitle className="text-base">Analytics</CardTitle>
            </CardHeader>
            <CardContent className="grid md:grid-cols-2 gap-4">
              <div className="h-40">
                <ResponsiveContainer width="100%" height="100%">
                  <BarChart data={categoryChart}>
                    <XAxis dataKey="name" />
                    <YAxis allowDecimals={false} />
                    <Tooltip />
                    <Bar dataKey="value" radius={[8, 8, 0, 0]} />
                  </BarChart>
                </ResponsiveContainer>
                <div className="text-xs text-muted-foreground mt-1">Sentiment distribution</div>
              </div>
              <div className="h-40">
                <ResponsiveContainer width="100%" height="100%">
                  <PieChart>
                    <Pie data={priorityChart} dataKey="value" nameKey="name" outerRadius={65}>
                      {priorityChart.map((_, idx) => (
                        <Cell key={idx} />
                      ))}
                    </Pie>
                    <Tooltip />
                  </PieChart>
                </ResponsiveContainer>
                <div className="text-xs text-muted-foreground mt-1">Priority split</div>
              </div>
            </CardContent>
          </Card>

          {selected && (
            <Card className="rounded-2xl">
              <CardHeader>
                <div className="flex items-center justify-between">
                  <CardTitle className="text-base">Email Details</CardTitle>
                  <div className="flex gap-2">
                    <Badge variant="secondary" className="rounded-full">{selected.priority}</Badge>
                    <Badge className="rounded-full" variant={selected.sentiment === "Negative" ? "destructive" : selected.sentiment === "Positive" ? "default" : "secondary"}>{selected.sentiment}</Badge>
                  </div>
                </div>
                <div className="text-xs text-muted-foreground">Received: {new Date(selected.receivedAt).toLocaleString()}</div>
              </CardHeader>
              <CardContent className="space-y-4">
                <div className="space-y-1">
                  <div className="text-sm"><span className="font-medium">From:</span> {selected.from}</div>
                  <div className="text-sm"><span className="font-medium">Subject:</span> {selected.subject}</div>
                  <div className="text-sm"><span className="font-medium">Product:</span> {selected.product}</div>
                </div>

                <Tabs defaultValue="body" className="w-full">
                  <TabsList>
                    <TabsTrigger value="body">Body</TabsTrigger>
                    <TabsTrigger value="extracted">Extracted Info</TabsTrigger>
                    <TabsTrigger value="draft">Draft Reply</TabsTrigger>
                  </TabsList>
                  <TabsContent value="body">
                    <pre className="text-sm whitespace-pre-wrap bg-muted p-3 rounded-xl">{selected.body}</pre>
                  </TabsContent>
                  <TabsContent value="extracted">
                    <div className="grid grid-cols-2 gap-3 text-sm">
                      <div>
                        <div className="font-medium mb-1">Contact</div>
                        <div className="space-y-1">
                          {selected.info.phone.length === 0 && selected.info.emails.length === 0 && (
                            <div className="text-muted-foreground">None detected</div>
                          )}
                          {selected.info.phone.map((p, i) => (
                            <div key={i}>📞 {p}</div>
                          ))}
                          {selected.info.emails.map((e, i) => (
                            <div key={i}>✉️ {e}</div>
                          ))}
                        </div>
                      </div>
                      <div>
                        <div className="font-medium mb-1">Detected Intents</div>
                        <div className="flex flex-wrap gap-2">
                          {selected.info.intents.length === 0 ? (
                            <Badge variant="outline" className="rounded-full">General</Badge>
                          ) : (
                            selected.info.intents.map((t) => <Badge key={t} variant="outline" className="rounded-full">{t}</Badge>)
                          )}
                        </div>
                      </div>
                    </div>
                  </TabsContent>
                  <TabsContent value="draft">
                    <div className="space-y-2">
                      <Label>AI-generated Draft</Label>
                      <Textarea className="min-h-[160px]" value={selected.draft} onChange={(e) => setSelected({ ...selected, draft: e.target.value })} />
                      <div className="flex gap-2">
                        <Button><Send className="w-4 h-4 mr-1" />Send</Button>
                        <Button variant="outline">Save Draft</Button>
                      </div>
                    </div>
                  </TabsContent>
                </Tabs>
              </CardContent>
            </Card>
          )}
        </div>
      </main>

      <footer className="max-w-7xl mx-auto p-6 text-xs text-muted-foreground">
        Demo UI — Replace mock NLP with your backend (LLM + RAG) and wire the Sync button to your email ingestion job.
      </footer>
    </div>
  );
}

