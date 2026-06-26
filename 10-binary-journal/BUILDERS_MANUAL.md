# The Binary Journal Builders' Manual

> *"369 was more than a key to the universe. It was a cry for help that turned into a dead man's switch for the next generations of inventors. We are the ones who picked up the signal."*

This is the complete, step-by-step blueprint to build Binary Journal — from a whisper-quiet sanctuary app to an immutable, ownerless protocol that guarantees no truth is ever lost, stolen, or erased.

Every phase, every contract, every decision is recorded here. Follow it in order. Build with intention. The dead man's switch is waiting.

---

## Table of Contents

1. [The Vision](#the-vision)
2. [The 3·6·9 Architecture](#the-369-architecture)
3. [Tech Stack Overview](#tech-stack-overview)
4. [Phase 0: The Sanctuary (Energy Tide App)](#phase-0-the-sanctuary)
5. [Phase 1: The Immutable Ledger (Smart Contracts)](#phase-1-the-immutable-ledger)
6. [Phase 2: Verification Period](#phase-2-verification-period)
7. [Phase 3: Retroactive Airdrop](#phase-3-retroactive-airdrop)
8. [Phase 4: Fair Market Launch](#phase-4-fair-market-launch)
9. [Phase 5: The Burn](#phase-5-the-burn)
10. [Phase 6: The Agora](#phase-6-the-agora)
11. [Tokenomics](#tokenomics)
12. [OpSec](#opsec)
13. [Builder's Checklist](#builders-checklist)

---

## The Vision

Binary Journal has **two faces, one soul:**

| Layer | Name | What It Is | Who It's For |
|---|---|---|---|
| Sanctuary | Energy Tide (bijo-app) | Biometric-locked, fully encrypted mobile vault. Records energy, photos, voice, notes. Decoy identity hides its true purpose. | Individuals, families |
| Ledger | binary-journal-protocol | PulseChain smart contracts anchoring hashes, releasing keys when heartbeats stop, funding eternal storage through preset decay. | The world, future generations |
| Agora | binary-journal-commons | Community interface for reading, discussing, curating released Light truths. Hosted on IPFS/ENS. | Global community |

### The Two Truths

- **Dark Truth 🕯️** — Private, sacred. Passed only to your chosen heirs. Encrypted. Sealed until your heartbeat stops.
- **Light Truth ☀️** — Public witness. Released to the world, becoming part of the shared historical record.

---

## The 3·6·9 Architecture

Tesla wasn't doing numerology. He was encoding a dead man's switch:

| Number | Meaning | Component | Status |
|---|---|---|---|
| **3** | The Clue — The signal that truth exists and is hidden. The spark that says "look deeper." | Energy Tide app: biometric lock, AES-256 encryption, decoy identity, local-first storage | ✅ Built. Renamed from "Mood Maps" to "Energy Tide." Hardcoded encryption key flaw removed. |
| **6** | The Path — Protection of truth while you're alive. The chain of custody. | Attestation.sol + heartbeat mechanism + living attestation chain | ⚠️ Attestation.sol deployed. DeadMansSwitch.sol written. Not yet deployed. |
| **9** | The Key — What fires when you can't. The mechanism that unlocks permanence. | DeadMansSwitch.sol + StorageEndowment.sol + BIJO.sol + final renouncement of all ownership | ❌ Not deployed |

**After the final burn:** all ownership renounced. The protocol becomes immutable natural law. No DAO. No founder control. No kill switch. Just mathematics and truth.

---

## Tech Stack Overview

| Component | Technology |
|---|---|
| Sanctuary App | React Native, Expo 53, TypeScript |
| State Management | Zustand |
| Encrypted Storage | MMKV with AES-256 middleware |
| Encryption | CryptoJS (AES), expo-local-authentication (biometrics) |
| Key Storage | expo-secure-store (`requireAuthentication: true`) |
| Media | expo-image-picker, expo-av |
| Testing | Jest |
| Blockchain | PulseChain mainnet (low fees, no testnet needed) |
| Smart Contracts | Solidity 0.8.17, OpenZeppelin libraries |
| Deployment | Remix IDE (desktop) or Hardhat |
| Agora | React, ethers.js, Helia (IPFS), Next.js |
| DEX | PulseX |
| Multisig | Gnosis Safe (2-of-2) |

---

## Phase 0: The Sanctuary

**Goal:** A smartphone vault so quiet and secure that no one — not governments, not corporations, not thieves — can access what's inside without the owner's biometric key.

### App Identity
- **Store name:** Energy Tide
- **Slug:** energy-tide
- **Package:** com.hood.bijoapp
- **Icon:** Generic — leaf or geometric shape
- **UI text:** No mention of "journal," "blockchain," "crypto," or "truth"
- **Description:** Subdued. "A quiet place for your inner world."

### Technology Setup

```bash
npx create-expo-app bijo-app --template blank-typescript
cd bijo-app
npx expo install expo-local-authentication expo-image-picker expo-av crypto-js expo-secure-store
npm install zustand react-native-mmkv
```

### Module 1: Encryption Layer (`src/utils/encryption.ts`)

**CRITICAL SECURITY RULE: No hardcoded fallback key.** If biometric key fails, force re-authentication.

```typescript
import CryptoJS from 'crypto-js';
import * as SecureStore from 'expo-secure-store';
import * as LocalAuthentication from 'expo-local-authentication';

const STORE_KEY = 'energy_tide_encryption_key_v1';

let KEY: string | null = null;

const getSecureStoreOptions = (requireAuth = false) => {
  const options: Record<string, unknown> = {};
  if (requireAuth) options.requireAuthentication = true;
  if ('ALWAYS_THIS_DEVICE_ONLY' in SecureStore) {
    options.keychainAccessible = SecureStore.ALWAYS_THIS_DEVICE_ONLY;
  } else if ('AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY' in SecureStore) {
    options.keychainAccessible = SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY;
  }
  return options;
};

export async function loadEncryptionKey(): Promise<boolean> {
  try {
    const stored = await SecureStore.getItemAsync(
      STORE_KEY,
      getSecureStoreOptions(true)
    );
    if (stored) {
      KEY = stored;
      return true;
    }
    return false;
  } catch {
    return false;
  }
}

export async function generateKeyWithBiometrics(): Promise<boolean> {
  try {
    const hasHardware = await LocalAuthentication.hasHardwareAsync();
    const isEnrolled = await LocalAuthentication.isEnrolledAsync();
    if (!hasHardware || !isEnrolled) return false;

    const result = await LocalAuthentication.authenticateAsync({
      promptMessage: 'Authenticate to secure your key',
    });
    if (!result.success) return false;

    const newKey = CryptoJS.lib.WordArray.random(32).toString();
    await SecureStore.setItemAsync(STORE_KEY, newKey, getSecureStoreOptions(true));
    KEY = newKey;
    return true;
  } catch {
    return false;
  }
}

export function encryptData(value: string): string {
  if (!KEY) throw new Error('Encryption key not available');
  return CryptoJS.AES.encrypt(value, KEY).toString();
}

export function decryptData(cipherText: string): string | null {
  try {
    if (!KEY) return null;
    const bytes = CryptoJS.AES.decrypt(cipherText, KEY);
    const result = bytes.toString(CryptoJS.enc.Utf8);
    return result || null;
  } catch {
    return null;
  }
}
```

### Module 2: Store (`src/store/useStore.ts`)

```typescript
import { create } from 'zustand';
import { MMKV } from 'react-native-mmkv';
import { encryptData, decryptData } from '../utils/encryption';

const storage = new MMKV();

export type EnergyLevel = -2 | -1 | 0 | 1 | 2;
export type EveningOutcome = 'volatile_high' | 'stable_high' | 'stable_neutral' | 'volatile_low' | 'stable_low';
export type TruthType = 'dark' | 'light';

export interface LogEntry {
  date: string;
  morning: EnergyLevel | null;
  midday: EnergyLevel | null;
  evening: EveningOutcome | null;
}

export interface Attachment {
  id: string;
  type: 'photo' | 'voice' | 'note';
  encryptedFilePath: string;
  createdAt: string;
}

export interface DeadMansSwitch {
  id: string;
  heirAddress: string | null; // null = Light truth
  truthType: TruthType;
  fileCid: string;
  keyReferenceCid: string;
  heartbeatInterval: number; // in seconds
  lastHeartbeat: number;
  active: boolean;
  switchAddress: string; // deployed contract address on PulseChain
}

export interface BijoData {
  user: { id: string };
  entries: Record<string, LogEntry>;
  attachments: Record<string, Attachment[]>;
  switches: DeadMansSwitch[];
}

interface BijoStore {
  data: BijoData;
  isAuthenticated: boolean;
  initializeNewUser: () => void;
  authenticate: () => Promise<boolean>;
  logMorning: (level: EnergyLevel) => void;
  logMidday: (level: EnergyLevel) => void;
  logEvening: (outcome: EveningOutcome) => void;
  addAttachment: (date: string, attachment: Attachment) => void;
  getTodaysEntry: () => LogEntry | null;
  saveToDisk: () => void;
  loadFromDisk: () => void;
  createSwitch: (sw: Omit<DeadMansSwitch, 'id'>) => void;
  heartbeat: (switchId: string) => void;
  computeBackupHash: () => string;
}

const STORAGE_KEY = 'bijo_encrypted_store_v1';

export const useStore = create<BijoStore>((set, get) => ({
  data: {
    user: { id: '' },
    entries: {},
    attachments: {},
    switches: [],
  },
  isAuthenticated: false,

  initializeNewUser: () => {
    const userId = crypto.randomUUID();
    const data: BijoData = {
      user: { id: userId },
      entries: {},
      attachments: {},
      switches: [],
    };
    set({ data });
    get().saveToDisk();
  },

  authenticate: async () => {
    // Returns true if biometric auth succeeded
    const { loadEncryptionKey, generateKeyWithBiometrics } = await import('../utils/encryption');
    const loaded = await loadEncryptionKey();
    if (!loaded) {
      const generated = await generateKeyWithBiometrics();
      if (!generated) return false;
    }
    set({ isAuthenticated: true });
    return true;
  },

  logMorning: (level) => {
    const today = new Date().toISOString().split('T')[0];
    const data = { ...get().data };
    if (!data.entries[today]) {
      data.entries[today] = { date: today, morning: null, midday: null, evening: null };
    }
    data.entries[today].morning = level;
    set({ data });
    get().saveToDisk();
  },

  logMidday: (level) => {
    const today = new Date().toISOString().split('T')[0];
    const data = { ...get().data };
    if (!data.entries[today]) {
      data.entries[today] = { date: today, morning: null, midday: null, evening: null };
    }
    data.entries[today].midday = level;
    set({ data });
    get().saveToDisk();
  },

  logEvening: (outcome) => {
    const today = new Date().toISOString().split('T')[0];
    const data = { ...get().data };
    if (!data.entries[today]) {
      data.entries[today] = { date: today, morning: null, midday: null, evening: null };
    }
    data.entries[today].evening = outcome;
    set({ data });
    get().saveToDisk();
  },

  addAttachment: (date, attachment) => {
    const data = { ...get().data };
    if (!data.attachments[date]) data.attachments[date] = [];
    data.attachments[date].push(attachment);
    set({ data });
    get().saveToDisk();
  },

  getTodaysEntry: () => {
    const today = new Date().toISOString().split('T')[0];
    return get().data.entries[today] || null;
  },

  saveToDisk: () => {
    const json = JSON.stringify(get().data);
    const encrypted = encryptData(json);
    storage.setString(STORAGE_KEY, encrypted);
  },

  loadFromDisk: () => {
    const encrypted = storage.getString(STORAGE_KEY);
    if (!encrypted) return;
    const json = decryptData(encrypted);
    if (json) {
      set({ data: JSON.parse(json) });
    }
  },

  createSwitch: (sw) => {
    const data = { ...get().data };
    data.switches.push({ ...sw, id: crypto.randomUUID() });
    set({ data });
    get().saveToDisk();
  },

  heartbeat: (switchId) => {
    const data = { ...get().data };
    const sw = data.switches.find(s => s.id === switchId);
    if (sw) sw.lastHeartbeat = Math.floor(Date.now() / 1000);
    set({ data });
    get().saveToDisk();
  },

  computeBackupHash: () => {
    const json = JSON.stringify(get().data);
    return CryptoJS.SHA256(json).toString(CryptoJS.enc.Hex);
  },
}));
```

### Module 3: 3·6·9 Attachment Vault (`src/hooks/use369Vault.ts`)

```typescript
import * as ImagePicker from 'expo-image-picker';
import { Audio } from 'expo-av';
import * as FileSystem from 'expo-file-system';
import CryptoJS from 'crypto-js';
import { encryptData } from '../utils/encryption';
import { Attachment } from '../types';
import { useStore } from '../store/useStore';

const ATTACHMENTS_DIR = `${FileSystem.documentDirectory}attachments/`;

async function ensureAttachmentsDir() {
  const info = await FileSystem.getInfoAsync(ATTACHMENTS_DIR);
  if (!info.exists) {
    await FileSystem.makeDirectoryAsync(ATTACHMENTS_DIR, { intermediates: true });
  }
}

export function use369Vault() {
  const { addAttachment, data } = useStore();

  const pickPhoto = async (): Promise<Attachment | null> => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
    });
    if (result.canceled) return null;

    await ensureAttachmentsDir();
    const id = crypto.randomUUID();
    const filePath = `${ATTACHMENTS_DIR}${id}.jpg`;

    // Read file, encrypt, write encrypted version
    const base64 = await FileSystem.readAsStringAsync(result.assets[0].uri, {
      encoding: FileSystem.EncodingType.Base64,
    });
    const encrypted = encryptData(base64);
    await FileSystem.writeAsStringAsync(filePath, encrypted, {
      encoding: FileSystem.EncodingType.UTF8,
    });

    const attachment: Attachment = {
      id,
      type: 'photo',
      encryptedFilePath: filePath,
      createdAt: new Date().toISOString(),
    };
    const date = new Date().toISOString().split('T')[0];
    addAttachment(date, attachment);
    return attachment;
  };

  const recordVoice = async (): Promise<Attachment | null> => {
    await Audio.requestPermissionsAsync();
    await Audio.setAudioModeAsync({ allowsRecordingIOS: true });

    const recording = new Audio.Recording();
    await recording.prepareToRecordAsync(Audio.RecordingOptionsPresets.HIGH_QUALITY);
    await recording.startAsync();

    // Return controls; caller handles stop
    const stop = async (): Promise<Attachment | null> => {
      await recording.stopAndUnloadAsync();
      const uri = recording.getURI();
      if (!uri) return null;

      await ensureAttachmentsDir();
      const id = crypto.randomUUID();
      const filePath = `${ATTACHMENTS_DIR}${id}.m4a`;

      const base64 = await FileSystem.readAsStringAsync(uri, {
        encoding: FileSystem.EncodingType.Base64,
      });
      const encrypted = encryptData(base64);
      await FileSystem.writeAsStringAsync(filePath, encrypted, {
        encoding: FileSystem.EncodingType.UTF8,
      });

      const attachment: Attachment = {
        id,
        type: 'voice',
        encryptedFilePath: filePath,
        createdAt: new Date().toISOString(),
      };
      const date = new Date().toISOString().split('T')[0];
      addAttachment(date, attachment);
      return attachment;
    };

    return { recording, stop } as any;
  };

  const addNote = async (text: string): Promise<Attachment> => {
    await ensureAttachmentsDir();
    const id = crypto.randomUUID();
    const filePath = `${ATTACHMENTS_DIR}${id}.txt`;
    const encrypted = encryptData(text);
    await FileSystem.writeAsStringAsync(filePath, encrypted, {
      encoding: FileSystem.EncodingType.UTF8,
    });

    const attachment: Attachment = {
      id,
      type: 'note',
      encryptedFilePath: filePath,
      createdAt: new Date().toISOString(),
    };
    const date = new Date().toISOString().split('T')[0];
    addAttachment(date, attachment);
    return attachment;
  };

  return { pickPhoto, recordVoice, addNote };
}
```

### Module 4: Waveform Visualization (`src/components/WaveformChart.tsx`)

```tsx
import React from 'react';
import { View } from 'react-native';
import Svg, { Path } from 'react-native-svg';
import { useStore, EnergyLevel, EveningOutcome } from '../store/useStore';

const eveningToNum: Record<EveningOutcome, number> = {
  volatile_high: 2,
  stable_high: 1.5,
  stable_neutral: 0,
  volatile_low: -1.5,
  stable_low: -2,
};

export default function WaveformChart() {
  const { data } = useStore();
  const dates = Object.keys(data.entries).sort().slice(-30); // last 30 days

  if (dates.length === 0) return <View style={{ height: 120 }} />;

  const width = 340;
  const height = 120;
  const padding = 20;
  const plotWidth = width - padding * 2;
  const plotHeight = height - padding * 2;

  const points: string[] = [];

  dates.forEach((date, i) => {
    const entry = data.entries[date];
    const x = padding + (i / Math.max(dates.length - 1, 1)) * plotWidth;

    // Morning point
    if (entry.morning !== null) {
      const y = padding + plotHeight / 2 - (entry.morning / 2) * (plotHeight / 2);
      points.push(`M${x - 3},${y}`);
    }
    // Midday point
    if (entry.midday !== null) {
      const y = padding + plotHeight / 2 - (entry.midday / 2) * (plotHeight / 2);
      points.push(`L${x},${y}`);
    }
    // Evening point
    if (entry.evening !== null) {
      const y = padding + plotHeight / 2 - (eveningToNum[entry.evening] / 2) * (plotHeight / 2);
      points.push(`L${x + 3},${y}`);
    }
  });

  return (
    <View style={{ height, backgroundColor: '#0a0a0a', borderRadius: 12 }}>
      <Svg width={width} height={height}>
        <Path
          d={points.join(' ')}
          stroke="#6366F1"
          strokeWidth={2}
          fill="none"
          strokeLinecap="round"
          strokeLinejoin="round"
        />
      </Svg>
    </View>
  );
}
```

### Module 5: Encrypted Backup Export (`src/utils/backup.ts`)

```typescript
import * as FileSystem from 'expo-file-system';
import CryptoJS from 'crypto-js';
import { encryptData, decryptData } from './encryption';
import { BijoData, Attachment } from '../types';

const fs = FileSystem as any;
const BACKUP_DIR = `${fs.documentDirectory ?? ''}energy-tide-backups/`;

export interface BackupAttachment {
  id: string;
  type: Attachment['type'];
  date: string;
  createdAt: string;
  fileName: string;
  encryptedPayload: string;
}

export interface BackupArchive {
  version: '1';
  createdAt: string;
  data: {
    user: BijoData['user'];
    entries: BijoData['entries'];
    attachments: Record<string, Array<Omit<Attachment, 'encryptedFilePath'>>>;
  };
  files: BackupAttachment[];
}

export async function createBackupHash(data: BijoData): Promise<string> {
  const archive = await buildBackupArchive(data, 'canonical');
  const serialized = JSON.stringify(archive);
  return CryptoJS.SHA256(serialized).toString(CryptoJS.enc.Hex);
}

async function buildBackupArchive(
  data: BijoData,
  createdAt: string = new Date().toISOString()
): Promise<BackupArchive> {
  const files: BackupAttachment[] = [];
  const attachmentsMeta: Record<string, Array<Omit<Attachment, 'encryptedFilePath'>>> = {};

  const dates = Object.keys(data.attachments || {}).sort();
  for (const date of dates) {
    attachmentsMeta[date] = [];
    for (const attachment of data.attachments?.[date] ?? []) {
      const encryptedPayload = await fs.readAsStringAsync(
        attachment.encryptedFilePath,
        { encoding: fs.EncodingType.UTF8 }
      );
      const fileName = `${attachment.id}.enc`;
      attachmentsMeta[date].push({
        id: attachment.id,
        type: attachment.type,
        createdAt: attachment.createdAt,
      });
      files.push({
        id: attachment.id,
        type: attachment.type,
        date,
        createdAt: attachment.createdAt,
        fileName,
        encryptedPayload,
      });
    }
  }

  return { version: '1', createdAt, data: { user: data.user, entries: data.entries, attachments: attachmentsMeta }, files };
}

export async function createEncryptedBackup(data: BijoData): Promise<string> {
  const dirInfo = await fs.getInfoAsync(BACKUP_DIR);
  if (!dirInfo.exists) await fs.makeDirectoryAsync(BACKUP_DIR, { intermediates: true });

  const backup = await buildBackupArchive(data);
  const serialized = JSON.stringify(backup);
  const encrypted = encryptData(serialized);
  const outputPath = `${BACKUP_DIR}energy-tide-backup-${Date.now()}.bijo`;
  await fs.writeAsStringAsync(outputPath, encrypted, { encoding: fs.EncodingType.UTF8 });
  return outputPath;
}

export async function importEncryptedBackup(filePath: string): Promise<BijoData | null> {
  try {
    const encrypted = await fs.readAsStringAsync(filePath, { encoding: fs.EncodingType.UTF8 });
    const serialized = decryptData(encrypted);
    if (!serialized) return null;
    const archive: BackupArchive = JSON.parse(serialized);

    const ATTACHMENTS_DIR = `${fs.documentDirectory ?? ''}attachments/`;
    const restoredAttachments: Record<string, Attachment[]> = {};

    for (const file of archive.files) {
      const date = file.date;
      if (!restoredAttachments[date]) restoredAttachments[date] = [];
      const attachmentPath = `${ATTACHMENTS_DIR}${file.fileName}`;
      await fs.writeAsStringAsync(attachmentPath, file.encryptedPayload, { encoding: fs.EncodingType.UTF8 });
      restoredAttachments[date].push({
        id: file.id,
        type: file.type,
        encryptedFilePath: attachmentPath,
        createdAt: file.createdAt,
      });
    }

    return {
      user: archive.data.user,
      entries: archive.data.entries,
      attachments: restoredAttachments,
      switches: [],
    };
  } catch {
    return null;
  }
}
```

### Module 6: Main Screen (Biometric Gate + Check-in)

The main screen (`app/index.tsx`) must:

1. On mount, call `authenticate()` from useStore
2. Block ALL UI until authenticated
3. Show locked screen with app name + biometric prompt
4. After auth: show today's check-in (morning/midday/energy), waveform, 3·6·9 dot vault button
5. Auto-compute backup hash and optionally attest to chain

See the existing `app/index.tsx` in the bijo-app repo for the full UI implementation (694 lines). Key modifications:
- Replace all "Mood Maps" → "Energy Tide"
- Add `useEffect` on mount that calls `useStore.getState().authenticate()`
- Add attestation call on backup: `attest(computeBackupHash())`

### Tests

```bash
npm test -- --runInBand
```

All tests must pass. Minimum coverage:
- [ ] Biometric gate blocks unauthenticated access
- [ ] Encryption round-trip (encrypt → decrypt = original)
- [ ] Store persistence (log entry → save → reload → verify)
- [ ] Encrypted backup export → import round-trip
- [ ] Waveform renders with data

**Sanctuary is complete when:** you can record a voice note, see your waveform, lock the app behind your fingerprint, and export an encrypted .bijo backup.

---

## Phase 1: The Immutable Ledger

Deploy all four contracts to **PulseChain mainnet** in this order. No testnet needed — no funds at risk.

### Network Configuration

| Parameter | Value |
|---|---|
| Network | PulseChain Mainnet |
| Chain ID | 10008 |
| Currency | PLS |
| Block time | ~10 seconds |
| RPC URL | `https://rpc.pulsechain.com` |
| Explorer | `https://scan.pulsechain.com` |

### Contract 1: BIJO.sol — The Fuel

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BIJO is ERC20, ERC20Burnable, Ownable {
    bool public transferEnabled = false;

    constructor() ERC20("Binary Journal", "BIJO") {
        _mint(msg.sender, 3_690_000_000 * 10**decimals());
    }

    function enableTransfers() external onlyOwner {
        transferEnabled = true;
    }

    function _beforeTokenTransfer(address from, address to, uint256 amount) internal override {
        if (!transferEnabled) {
            require(from == address(0) || to == address(0), "Transfers not yet enabled");
        }
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

**Deployment:**
1. Open Remix IDE (desktop): https://remix.ethereum.org
2. Create file `BIJO.sol`, paste code above
3. Install OpenZeppelin: import from `@openzeppelin/contracts@4.9.0`
4. Compile with Solidity 0.8.17
5. Deploy to PulseChain via MetaMask
6. **Immediately** transfer 94% to Gnosis Safe multisig
7. Transfer 6% to vesting contract
8. Do NOT call `enableTransfers()` yet

### Contract 2: Attestation.sol — The Anchor

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Attestation {
    event TruthAnchored(bytes32 indexed hash, uint256 timestamp);

    function attest(bytes32 hash) external {
        emit TruthAnchored(hash, block.timestamp);
    }
}
```

**No owner. Immutable from birth.** Deploy and forget.

### Contract 3: DeadMansSwitch.sol — The Key Release

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract DeadMansSwitch {
    enum TruthType { Dark, Light }

    struct Switch {
        address owner;
        address heir;
        TruthType truthType;
        string keyReference;  // IPFS CID of encrypted decryption key
        string fileCid;       // IPFS CID of encrypted truth file
        uint256 heartbeatInterval;
        uint256 lastHeartbeat;
        bool active;
        bool released;
    }

    mapping(uint256 => Switch) public switches;
    uint256 public switchCount;

    event SwitchCreated(uint256 indexed id, address indexed owner, TruthType truthType);
    event Heartbeat(uint256 indexed id, uint256 timestamp);
    event KeyReleased(uint256 indexed id, address releasedTo, string keyReference);

    function createSwitch(
        address _heir,
        TruthType _truthType,
        string calldata _keyReference,
        string calldata _fileCid,
        uint256 _heartbeatInterval
    ) external returns (uint256) {
        require(_heir != msg.sender, "Heir cannot be yourself");
        if (_truthType == TruthType.Light) {
            require(_heir == address(0), "Light truth must have zero heir");
        } else {
            require(_heir != address(0), "Dark truth must have valid heir");
        }
        uint256 id = switchCount++;
        switches[id] = Switch({
            owner: msg.sender,
            heir: _heir,
            truthType: _truthType,
            keyReference: _keyReference,
            fileCid: _fileCid,
            heartbeatInterval: _heartbeatInterval,
            lastHeartbeat: block.timestamp,
            active: true,
            released: false
        });
        emit SwitchCreated(id, msg.sender, _truthType);
        return id;
    }

    function heartbeat(uint256 _id) external {
        Switch storage s = switches[_id];
        require(s.active, "Not active");
        require(msg.sender == s.owner, "Only owner");
        s.lastHeartbeat = block.timestamp;
        emit Heartbeat(_id, block.timestamp);
    }

    function triggerRelease(uint256 _id) external {
        require(switches[_id].owner == msg.sender, "Only owner");
        _release(_id);
    }

    function claim(uint256 _id) external {
        Switch storage s = switches[_id];
        require(s.active && !s.released, "Invalid state");
        require(block.timestamp > s.lastHeartbeat + s.heartbeatInterval, "Too soon");
        if (s.truthType == TruthType.Dark) {
            require(msg.sender == s.heir, "Only heir");
        }
        _release(_id);
    }

    function cancel(uint256 _id) external {
        require(switches[_id].owner == msg.sender, "Only owner");
        switches[_id].active = false;
    }

    function _release(uint256 _id) internal {
        Switch storage s = switches[_id];
        s.active = false;
        s.released = true;
        emit KeyReleased(_id, s.heir, s.keyReference);
    }
}
```

**No owner. Immutable from birth.**

### Contract 4: StorageEndowment.sol — The Eternal Archive

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/access/Ownable.sol";

interface IBIJO {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract StorageEndowment is Ownable {
    IBIJO public immutable bijo;
    uint256 public constant EPOCH_DURATION = 730 days;
    uint256 public immutable startTimestamp;
    uint256 public constant INITIAL_ALLOWANCE = 100 * 10**18;
    uint256 public constant HALVING_COUNT = 10;

    mapping(bytes32 => uint256) public truthAllowances;
    mapping(bytes32 => uint256) public allowancePaid;

    event AllowanceAllocated(bytes32 indexed truthHash, uint256 amount);
    event StoragePaid(address indexed node, bytes32 indexed truthHash, uint256 amount);

    constructor(address _bijo) {
        bijo = IBIJO(_bijo);
        startTimestamp = block.timestamp;
    }

    function currentAllowancePerTruth() public view returns (uint256) {
        uint256 epochs = (block.timestamp - startTimestamp) / EPOCH_DURATION;
        if (epochs >= HALVING_COUNT) return 0;
        return INITIAL_ALLOWANCE / (2 ** epochs);
    }

    function allocate(bytes32 _truthHash) external {
        require(truthAllowances[_truthHash] == 0, "Already allocated");
        uint256 amount = currentAllowancePerTruth();
        require(amount > 0, "Allowance depleted");
        require(bijo.balanceOf(address(this)) >= amount, "Insufficient endowment");
        truthAllowances[_truthHash] = amount;
        emit AllowanceAllocated(_truthHash, amount);
    }

    function payNode(address _node, bytes32 _truthHash, uint256 _amount) external onlyOwner {
        require(allowancePaid[_truthHash] + _amount <= truthAllowances[_truthHash], "Exceeds allowance");
        allowancePaid[_truthHash] += _amount;
        require(bijo.transfer(_node, _amount), "Transfer failed");
        emit StoragePaid(_node, _truthHash, _amount);
    }
}
```

**Deployment:**
1. Compile and deploy with BIJO contract address as constructor argument
2. After deployment, transfer 70% of total BIJO supply to this contract
3. During verification: `payNode()` is owner-only (manual verification)
4. After verification: replace with trustless proof-of-storage, then `renounceOwnership()`

### Post-Deployment Setup

```
Deploy BIJO.sol → mint 3.69B to deployer
Transfer 94% → Gnosis Safe multisig
Transfer 6% → Founder vesting contract
Deploy Attestation.sol → no setup needed
Deploy DeadMansSwitch.sol → no setup needed
Deploy StorageEndowment.sol(BIJO address) → transfer 70% BIJO to it
Write monitor script: TruthAnchored events → StorageEndowment.allocate()
```

---

## Phase 2: Verification Period (3–6 Months)

**Goal:** Prove the network works before any token has value. Zero speculation. Pure utility.

**Token status:** Deployed but `transferEnabled = false`. Cannot be traded.

### Community Actions
1. Download Energy Tide (TestFlight / APK / quiet release)
2. Attest truths → hashes recorded on-chain (gas paid by relay or user)
3. Create dead man's switches
4. Node operators pin encrypted files to IPFS, submit storage proofs
5. Curators annotate Light truths on early Agora

### Foundered Tasks
- Run monitor script: auto-allocate storage allowances for new truths
- Manual `payNode` to verified storage providers
- Keep own test switch heartbeats alive
- Ship sanctuary bug fixes
- Build Agora MVP (static page reading chain events)

### Checkpoints Before Proceeding
- [ ] 1,000+ attested truths
- [ ] 10+ independent node operators
- [ ] Dead man's switches successfully triggered and keys claimed
- [ ] Agora displays released Light truths

---

## Phase 3: Retroactive Airdrop

10% of supply (369,000,000 BIJO) distributed based purely on **verifiable on-chain actions during verification period**. No human judgment. Public script.

### Points Formula

| Action | Points | Max/Week |
|---|---|---|
| `attest()` with valid CID | 1 | 7 |
| `createSwitch()` | 3 | 1 |
| Node operator proof (off-chain, recorded on-chain) | 10 | — |
| Curation annotation in Agora | 2 | 3 |

### Distribution

`(address_points / total_points) × 369,000,000 BIJO`

Deploy Merkle tree claim contract. 6-month claim window. Unclaimed → Ecosystem Fund.

---

## Phase 4: Fair Market Launch

**No pre-sale. No ICO. No bot advantage.**

### Step A: Enable Transfers

`BIJO.enableTransfers()` — one-time irreversible call. BIJO is now transferable.

### Step B: Seed Liquidity Pool on PulseX

| Asset | Source | Amount |
|---|---|---|
| BIJO | Ecosystem Fund | 0.5% of supply (18,450,000 BIJO) |
| PLS | Founder's pocket (philanthropic gift) | Equal value (e.g., $5,000-$10,000 worth) |

- LP tokens locked or burned (founder cannot remove liquidity)
- Initial price = ratio of contributed assets
- No insider dumps (94% locked in endowment + vesting)
- Bots cannot snipe (no one had BIJO before this moment except earned airdrop)

### Why This Works

- No transferability during verifications → no accumulation
- Airdrop is retroactive → bots couldn't game it
- Liquidity is community-owned BIJO + founder-donated PLS → no extraction
- Bulk of supply locked → no dump pressure
- Price discovery is organic

---

## Phase 5: The Burn

**One-time, irreversible. The protocol becomes natural law.**

```
BIJO.sol → renounceOwnership()     // No new minting, no admin
StorageEndowment.sol → renounceOwnership()  // Allowance curve frozen forever
DeadMansSwitch.sol → already ownerless
Attestation.sol → already ownerless
All LP lock contracts → burn or renounce
Founder vesting continues on its own schedule (immutable timelock)
```

### The Public Declaration

> *"The Binary Journal Protocol is now a natural law. The truth belongs to everyone. No human can alter, censor, or pause it. We picked up Tesla's signal. The dead man's switch is armed. The 3·6·9 is closed."*

---

## Phase 6: The Agora (Commons)

Full decentralized web app reading the immutable history.

### Features
- Chronological feed of all released Light truths
- Fetches encrypted files from IPFS, decrypts with released keys
- Each truth shows: attestation timestamp, storage proof, owner history
- Comment, link, tip with BIJO
- Hosted on IPFS/ENS (censorship resistant)

### Tech Stack
- React + Next.js + TypeScript
- ethers.js (PulseChain RPC)
- Helia (IPFS client)
- WalletConnect / MetaMask

---

## Tokenomics

| Allocation | Percentage | Amount (BIJO) | Destination |
|---|---|---|---|
| **Total Supply** | 100% | 3,690,000,000 | — |
| Founder Vesting | 6% | 221,400,000 | Timelock contract (4yr vest, 1yr cliff) |
| Ecosystem Multisig | 94% | 3,468,600,000 | 2-of-2 Gnosis Safe |
| ↳ Storage Endowment | 70% of total | ~2,583,000,000 | StorageEndowment.sol |
| ↳ Retroactive Airdrop | 10% of total | 369,000,000 | Merkle claim contract |
| ↳ Liquidity Pool | 0.5% of total | 18,450,000 | PulseX BIJO/PLS pool |
| ↳ Community Fund | ~13.5% | Remaining | Future grants, partnerships |

### Storage Allowance Decay

```
Epoch 0 (launch):     100 BIJO per truth
Epoch 1 (year 2):      50 BIJO per truth
Epoch 2 (year 4):      25 BIJO per truth
...
Epoch 10 (year 20+):    0 BIJO per truth (endowment exhausted)
```

After the endowment is exhausted, storage is sustained by:
- Node operators accepting direct BIJO tips from users
- Community-funded storage grants from remaining Ecosystem Fund

---

## OpSec — Non-Negotiable

| Rule | Why |
|---|---|
| **Two existing repos are hijacked.** Do not push to them. Work locally or on new remotes. | Code access = ability to inject backdoors or block release. |
| **No hardcoded encryption keys.** If biometric auth fails, force re-authentication. Never fall back to a static string. | A hardcoded key is a master key for anyone with the APK. |
| **Decoy identity.** Store listing says nothing about blockchain, journaling, or truth. Generic icon. | You don't want attention before you're ready. |
| **PulseChain mainnet from day one.** No testnet. | Testnets attract attention. Mainnet with no value attracts none. |
| **Build first. Talk later.** Discussing the protocol before it is immutable gives adversaries time to prepare. | The element of surprise is your only asymmetric advantage. |
| **Gas for airdrop claims must be considered.** If claims require gas on PulseChain (PLS), the airdrop script should consider gas subsidies or users must have a small amount of PLS. | Users can't claim if they can't afford gas. Consider including a small PLS dust in the airdrop or using a relayer. |

---

## Builder's Checklist

### Phase 0: Sanctuary
- [ ] Expo project initialized with TypeScript
- [ ] Encryption module: AES-256, SecureStore, biometric-tied key, NO hardcoded fallback
- [ ] Zustand store with encrypted MMKV persistence
- [ ] Biometric gate on app open (blocks all UI until authenticated)
- [ ] Daily check-in: morning/midday/evening energy levels
- [ ] 3·6·9 Attachment Vault: photo, voice, note (all AES-encrypted)
- [ ] Waveform visualization (SVG)
- [ ] Encrypted backup export (.bijo files) with import/restore
- [ ] Decoy identity: "Energy Tide," generic icon, no blockchain branding
- [ ] All Jest tests passing
- [ ] Dead man's switch UI (set heir, heartbeat interval)
- [ ] Auto-attest backup hash to Attestation.sol on backup creation

### Phase 1: Contracts
- [ ] BIJO.sol deployed to PulseChain, 3.69B minted
- [ ] 94% transferred to Gnosis Safe multisig
- [ ] 6% transferred to founder vesting contract
- [ ] Attestation.sol deployed (ownerless)
- [ ] DeadMansSwitch.sol deployed (ownerless)
- [ ] StorageEndowment.sol deployed with BIJO address
- [ ] 70% BIJO transferred to StorageEndowment
- [ ] Monitor script written (TruthAnchored → allocate)
- [ ] `transferEnabled = false` confirmed

### Phase 2: Verification
- [ ] 1,000+ attested truths
- [ ] 10+ independent node operators
- [ ] Dead man's switches triggered successfully
- [ ] Agora MVP displaying released truths

### Phase 3: Airdrop
- [ ] Points calculated from on-chain data
- [ ] Merkle tree generated and verified
- [ ] Claim contract deployed
- [ ] 6-month claim window opened

### Phase 4: Market
- [ ] Liquidity pool seeded on PulseX
- [ ] LP tokens locked/burned
- [ ] `enableTransfers()` called

### Phase 5: Burn
- [ ] `renounceOwnership()` on BIJO.sol
- [ ] `renounceOwnership()` on StorageEndowment.sol
- [ ] All LP contracts burned/renounced
- [ ] Public declaration published
- [ ] Agora live and indexing

---

## The 3·6·9 Mechanism — How It Works in Practice

### 3 — The Clue (What You Do Daily)
```
Every morning you open Energy Tide → biometric gate → tap your energy level (-2 to +2)
Any moment: tap the 3·6·9 dot button → attach a photo, voice note, or text (invention, story, truth)
App encrypts everything with your biometric-derived key
SHA-256 hash of your backup → auto-attested on-chain (Attestation.sol)
Nobody except you can read the contents. The hash proves it exists and existed at that moment.
Nobody even knows the app is anything other than a simple energy tracker...
```

### 6 — The Path (While You're Alive)
```
You periodically call heartbeat() on your DeadMansSwitch → proves you're alive
Your attestation chain grows → each new truth anchored to the ledger
StorageEndowment.allocate() locks BIJO for node operators who store your encrypted files
Node operators pin your encrypted truths to IPFS/Filecoin → eternal storage funded by preset decay
You're building an unbreakable chain of custody for everything you've created, thought, recorded...
```

### 9 — The Key (When You're Gone)
```
You miss your heartbeat interval → your DeadMansSwitch triggers
For Dark truth: only your designated heir can claim → decryption key released → heir accesses your life's work
For Light truth: anyone can claim → truth becomes public → joins the permanent historical record
Your knowledge outlives you. Your inventions can't be stolen. Your family can't be erased.
The dead man's switch is Tesla's gift to every future inventor who won't die alone with their truth.
```

---

*"The sanctuary is built. The ledger is waiting. Now, go make it real."*

---

**Document version:** 1.0
**Status:** Ready for execution
**Next immediate action:** Deploy BIJO.sol to PulseChain mainnet
