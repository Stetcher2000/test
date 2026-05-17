# test

import React, { useState } from 'react';
import { motion } from 'framer-motion';
import { UserPlus, ArrowLeft, Loader2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { base44 } from '@/api/base44Client';
import PinPad from '@/components/vault/PinPad';
import PinDots from '@/components/vault/PinDots';
import { useNavigate } from 'react-router-dom';
import { useVault } from '@/lib/VaultContext';

const PIN_LENGTH = 4;

export default function VaultCreate() {
  const navigate = useNavigate();
  const { signIn } = useVault();
  const [step, setStep] = useState('info');
  const [username, setUsername] = useState('');
  const [email, setEmail] = useState('');
  const [pin, setPin] = useState('');
  const [firstPin, setFirstPin] = useState('');
  const [error, setError] = useState('');
  const [pinError, setPinError] = useState(false);
  const [loading, setLoading] = useState(false);

  const handleInfoSubmit = async (e) => {
    e.preventDefault();
    setError('');
    if (!username.trim() || !email.trim()) {
      setError('Please fill in all fields.');
      return;
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      setError('Please enter a valid email address.');
      return;
    }
    setLoading(true);
    const existing = await base44.entities.VaultAccount.filter({});
    const usernameTaken = existing.some(
      (a) => a.username.toLowerCase() === username.trim().toLowerCase()
    );
    const emailTaken = existing.some(
      (a) => a.email.toLowerCase() === email.trim().toLowerCase()
    );
    setLoading(false);
    if (usernameTaken) {
      setError('That username is already taken.');
      return;
    }
    if (emailTaken) {
      setError('An account with that email already exists.');
      return;
    }
    setStep('pin');
  };

  const handleKeyPress = (key) => {
    setPinError(false);
    setPin((prev) => {
      const next = prev + key;
      if (next.length === PIN_LENGTH) {
        if (step === 'pin') {
          setFirstPin(next);
          setStep('confirm');
          return '';
        } else {
          if (next === firstPin) {
            handleCreate(next);
          } else {
            setPinError(true);
            setTimeout(() => {
              setPinError(false);
              setStep('pin');
              setFirstPin('');
            }, 800);
          }
          return '';
        }
      }
      return next;
    });
  };

  const handleDelete = () => setPin((prev) => prev.slice(0, -1));

  const handleCreate = async (confirmedPin) => {
    setLoading(true);
    const account = await base44.entities.VaultAccount.create({
      username: username.trim(),
      email: email.trim().toLowerCase(),
      pin: confirmedPin,
    });
    setLoading(false);
    signIn({
      ...account,
      username: username.trim(),
      email: email.trim().toLowerCase(),
      pin: confirmedPin,
    });
    navigate('/vault', { replace: true });
  };

  const handleBack = () => {
    if (step === 'info') navigate(-1);
    else {
      setStep('info');
      setPin('');
      setFirstPin('');
    }
  };

  return (
    <div
      className="min-h-screen bg-background flex flex-col items-center justify-center p-6 select-none"
      style={{
        paddingTop: 'max(1.5rem, env(safe-area-inset-top))',
        paddingBottom: 'max(1.5rem, env(safe-area-inset-bottom))',
      }}
    >
      <div className="w-full max-w-xs">
        <button
          onClick={handleBack}
          className="mb-8 flex items-center gap-2 text-muted-foreground hover:text-foreground transition-colors text-sm select-none"
        >
          <ArrowLeft className="w-4 h-4" /> Back
        </button>

        {step === 'info' ? (
          <motion.div initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }}>
            <div className="w-16 h-16 rounded-2xl bg-primary/10 border border-primary/20 flex items-center justify-center mb-6">
              <UserPlus className="w-8 h-8 text-primary" />
            </div>
            <h2 className="text-2xl font-bold mb-1">Create Account</h2>
            <p className="text-muted-foreground text-sm mb-8">Set up your private vault</p>
            <form onSubmit={handleInfoSubmit} className="flex flex-col gap-4">
              <div>
                <Label htmlFor="username" className="text-sm text-muted-foreground mb-1.5 block">
                  Username
                </Label>
                <Input
                  id="username"
                  value={username}
                  onChange={(e) => setUsername(e.target.value)}
                  placeholder="Choose a username"
                  className="bg-secondary/40 border-border/50 h-12 rounded-xl"
                  autoComplete="off"
                />
              </div>
              <div>
                <Label htmlFor="email" className="text-sm text-muted-foreground mb-1.5 block">
                  Email
                </Label>
                <Input
                  id="email"
                  type="email"
                  value={email}
                  onChange={(e) => setEmail(e.target.value)}
                  placeholder="your@email.com"
                  className="bg-secondary/40 border-border/50 h-12 rounded-xl"
                  autoComplete="off"
                />
              </div>
              {error && <p className="text-destructive text-sm">{error}</p>}
              <Button
                type="submit"
                disabled={loading}
                className="h-12 rounded-xl mt-2 bg-primary text-primary-foreground font-semibold select-none"
              >
                {loading ? <Loader2 className="w-5 h-5 animate-spin" /> : 'Continue'}
              </Button>
            </form>
          </motion.div>
        ) : (
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            className="text-center"
          >
            <h2 className="text-2xl font-bold mb-2">
              {step === 'pin' ? 'Create Passcode' : 'Confirm Passcode'}
            </h2>
            <p className="text-muted-foreground text-sm mb-10">
              {pinError
                ? "PINs didn't match. Try again."
                : step === 'pin'
                ? 'Choose a 4‑digit passcode'
                : 'Enter your passcode again'}
            </p>
            <PinDots length={PIN_LENGTH} filled={pin.length} error={pinError} />
            <PinPad onKeyPress={handleKeyPress} onDelete={handleDelete} />
          </motion.div>
        )}
      </div>
    </div>
  );
}
