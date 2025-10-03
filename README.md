<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kukirin G2 Pro - Interaktives UART Protokoll</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Roboto+Mono:wght@400;500&display=swap');
        .font-mono {
            font-family: 'Roboto Mono', monospace;
        }
        .nav-link.active {
            background-color: #3b82f6;
            color: white;
        }
        .content-section {
            display: none;
        }
        .content-section.active {
            display: block;
            animation: fadeIn 0.5s ease-in-out;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .byte-block {
            transition: all 0.2s ease-in-out;
            cursor: pointer;
        }
        .byte-block:hover {
            transform: translateY(-2px);
            box-shadow: 0 4px 10px rgba(0,0,0,0.1);
        }
        .byte-block.highlight {
             background-color: #3b82f6 !important;
             color: white !important;
        }
        .byte-block.highlight .byte-offset {
            color: #dbeafe !important;
        }
    </style>
    <!-- Chosen Palette: Tech Slate -->
    <!-- Application Structure Plan: A dashboard-style SPA with a fixed sidebar for navigation and a main content area. This structure allows users to easily jump between different aspects of the protocol (e.g., TX packets, RX packets, checksum calculation) without scrolling through a long document. Key interactions include interactive packet explorers where hovering/clicking bytes reveals their meaning, and a functional checksum calculator. This design prioritizes targeted information retrieval and practical application over linear reading, which is ideal for technical documentation. -->
    <!-- Visualization & Content Choices: Report Info -> Packet structure tables. Goal -> Organize/Inform. Viz -> Interactive byte stream visualization (HTML/JS) instead of static tables. Interaction -> Hover/click to show details. Justification -> More intuitive for protocol analysis. | Report Info -> Checksum algorithms. Goal -> Provide utility. Viz -> Checksum Calculator Tool (HTML/JS). Interaction -> User input for hex data, button click to calculate. Justification -> Turns documentation into a practical tool. | Report Info -> Driving modes table. Goal -> Compare/Inform. Viz -> Styled HTML table. Interaction -> Hover effects. Justification -> Table is the clearest format for this data. -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
</head>
<body class="bg-slate-50 text-slate-800">

    <div class="flex flex-col md:flex-row min-h-screen">
        <!-- Sidebar Navigation -->
        <aside class="w-full md:w-64 bg-white border-r border-slate-200 p-4 md:p-6 md:fixed md:h-full z-10">
            <h1 class="text-xl font-bold text-slate-900">Kukirin G2 Pro</h1>
            <h2 class="text-sm text-slate-500 mb-6">UART Protokoll Explorer</h2>
            <nav id="main-nav" class="flex flex-row md:flex-col space-x-2 md:space-x-0 md:space-y-2 overflow-x-auto pb-4">
                <a href="#" data-target="overview" class="nav-link active text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">Übersicht</a>
                <a href="#" data-target="tx-packets" class="nav-link text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">TX Pakete</a>
                <a href="#" data-target="rx-packets" class="nav-link text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">RX Pakete</a>
                <a href="#" data-target="checksum" class="nav-link text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">Checksum Rechner</a>
                <a href="#" data-target="sequences" class="nav-link text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">Start & Stopp</a>
                <a href="#" data-target="modes" class="nav-link text-left px-4 py-2 rounded-lg text-slate-700 hover:bg-blue-500 hover:text-white transition-colors duration-200">Fahrmodi</a>
            </nav>
        </aside>

        <!-- Main Content -->
        <main class="flex-1 md:ml-64 p-4 sm:p-6 md:p-10">
            
            <!-- Overview Section -->
            <section id="overview" class="content-section active">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">Protokoll Übersicht</h1>
                <p class="mb-8 text-slate-600">Dieses Dokument beschreibt das reverse-engineerte UART-Kommunikationsprotokoll zwischen dem Display (Master) und dem Motorcontroller (Slave) des Kukirin G2 Pro E-Scooters.</p>
                
                <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                        <h3 class="text-lg font-semibold mb-3">1. Physikalische Schnittstelle</h3>
                        <ul class="space-y-2 text-slate-700">
                            <li><strong class="font-medium text-slate-900">Protokoll:</strong> UART (asynchron)</li>
                            <li><strong class="font-medium text-slate-900">Baudrate:</strong> 9600 bps</li>
                            <li><strong class="font-medium text-slate-900">Datenformat:</strong> 8N1 (8 Datenbits, keine Parität, 1 Stoppbit)</li>
                            <li><strong class="font-medium text-slate-900">Spannung:</strong> 3.3V TTL</li>
                        </ul>
                    </div>
                    <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                         <h3 class="text-lg font-semibold mb-3">2. Kommunikationsstruktur</h3>
                        <p class="text-slate-700 mb-2"><strong class="font-medium text-slate-900">Zyklus-Intervall:</strong> ~60ms (~16-17 Hz)</p>
                        <p class="font-medium text-slate-900 mb-2">Sequenz-Muster:</p>
                        <div class="font-mono text-sm bg-slate-100 p-3 rounded-md text-slate-600">
                            [TX Burst] → [Pause] → [RX Burst] → [Idle]
                        </div>
                    </div>
                </div>
            </section>

            <!-- TX Packets Section -->
            <section id="tx-packets" class="content-section">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">TX Paket Explorer (Display → Controller)</h1>
                <p class="mb-8 text-slate-600">Klicke oder fahre über einen Block, um Details zu den Feldern des 20-Byte-Pakets anzuzeigen.</p>
                <div id="tx-packet-container" class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-5 lg:grid-cols-10 gap-2 font-mono text-center"></div>
                <div id="tx-packet-details" class="mt-6 bg-white p-6 rounded-xl shadow-sm border border-slate-100 min-h-[150px] flex items-center justify-center">
                    <p class="text-slate-500">Wähle einen Block aus, um dessen Beschreibung hier zu sehen.</p>
                </div>
            </section>

            <!-- RX Packets Section -->
            <section id="rx-packets" class="content-section">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">RX Paket Explorer (Controller → Display)</h1>
                <p class="mb-8 text-slate-600">Klicke oder fahre über einen Block, um Details zu den Feldern des 16-Byte-Pakets anzuzeigen.</p>
                <div id="rx-packet-container" class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-4 lg:grid-cols-8 gap-2 font-mono text-center"></div>
                <div id="rx-packet-details" class="mt-6 bg-white p-6 rounded-xl shadow-sm border border-slate-100 min-h-[150px] flex items-center justify-center">
                    <p class="text-slate-500">Wähle einen Block aus, um dessen Beschreibung hier zu sehen.</p>
                </div>
            </section>

            <!-- Checksum Section -->
            <section id="checksum" class="content-section">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">Checksum Rechner</h1>
                <p class="mb-8 text-slate-600">Berechne die CRC-Prüfsummen für TX- und RX-Pakete.</p>
                <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
                    <!-- TX Calculator -->
                    <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                        <h3 class="text-lg font-semibold mb-3">TX Checksum (CRC-16/MODBUS)</h3>
                        <p class="text-sm text-slate-500 mb-4">Gib die ersten 18 Bytes des TX-Pakets als Hex-String ein (z.B. `01 14 ... 00 00`).</p>
                        <textarea id="tx-checksum-input" rows="4" class="w-full p-2 border border-slate-300 rounded-md font-mono text-sm focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="01 14 01 02 0A 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00"></textarea>
                        <button id="calculate-tx-checksum" class="mt-4 w-full bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600 transition-colors">Berechnen</button>
                        <div class="mt-4">
                            <h4 class="font-semibold">Ergebnis:</h4>
                            <p id="tx-checksum-result" class="font-mono bg-slate-100 p-3 rounded-md text-slate-800 mt-2 break-all"></p>
                        </div>
                    </div>
                    <!-- RX Calculator -->
                    <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                        <h3 class="text-lg font-semibold mb-3">RX Checksum (CRC-8)</h3>
                        <p class="text-sm text-slate-500 mb-4">Gib die ersten 13 Bytes des RX-Pakets als Hex-String ein (z.B. `02 0E ... 00 00`).</p>
                        <textarea id="rx-checksum-input" rows="4" class="w-full p-2 border border-slate-300 rounded-md font-mono text-sm focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="02 0E 01 00 C0 00 00 00 0D AC 00 00 00"></textarea>
                        <button id="calculate-rx-checksum" class="mt-4 w-full bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600 transition-colors">Berechnen</button>
                        <div class="mt-4">
                            <h4 class="font-semibold">Ergebnis:</h4>
                            <p id="rx-checksum-result" class="font-mono bg-slate-100 p-3 rounded-md text-slate-800 mt-2 break-all"></p>
                        </div>
                    </div>
                </div>
            </section>
            
            <!-- Sequences Section -->
            <section id="sequences" class="content-section">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">Start- und Stoppsequenz</h1>
                <p class="mb-8 text-slate-600">Der Ablauf beim Ein- und Ausschalten des Scooters.</p>
                 <div class="space-y-8">
                    <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                        <h3 class="text-xl font-semibold mb-4 text-slate-800">5.1 Startvorgang (Handshake)</h3>
                        <p class="mb-4 text-slate-600">Der Startvorgang ist eine klassische Handshake-Prozedur, um sicherzustellen, dass der Controller betriebsbereit ist, bevor das Display seine volle Funktionalität anzeigt.</p>
                        <ol class="list-decimal list-inside space-y-3 text-slate-700">
                            <li><strong class="font-medium text-slate-900">Button-Druck:</strong> Der Nutzer drückt den Power-Button. Das Display wird mit Strom versorgt, bleibt aber dunkel und beginnt sofort, Standard-TX-Pakete zu senden.</li>
                            <li><strong class="font-medium text-slate-900">Controller-Antwort:</strong> Der Motorcontroller empfängt die TX-Pakete, initialisiert sich und beginnt, RX-Pakete zu senden.</li>
                            <li><strong class="font-medium text-slate-900">Handshake-Signal:</strong> Als Bestätigung sendet der Controller mindestens ein spezielles "Boot-Acknowledged"-Paket. Dieses ist daran zu erkennen, dass das <strong class="font-mono text-blue-600">Status-Flag</strong> an Offset 0x03 den Wert <strong class="font-mono text-blue-600">0x80</strong> hat.</li>
                            <li><strong class="font-medium text-slate-900">Display-Aktivierung:</strong> Das Display wartet auf dieses Paket. Nach Empfang ist der Handshake abgeschlossen und das Display schaltet die Benutzeroberfläche ein.</li>
                            <li><strong class="font-medium text-slate-900">Normalbetrieb:</strong> Der Controller sendet ab jetzt nur noch normale Pakete mit einem Status-Flag von <strong class="font-mono text-blue-600">0x00</strong>.</li>
                        </ol>
                    </div>
                     <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                        <h3 class="text-xl font-semibold mb-4 text-slate-800">5.2 Stoppvorgang (Timeout)</h3>
                         <p class="mb-4 text-slate-600">Das Ausschalten erfolgt nicht über ein spezielles Kommando, sondern durch einen Kommunikationsabbruch.</p>
                        <ol class="list-decimal list-inside space-y-3 text-slate-700">
                           <li><strong class="font-medium text-slate-900">Button-Druck (lang):</strong> Der Nutzer hält den Power-Button für mehrere Sekunden gedrückt.</li>
                           <li><strong class="font-medium text-slate-900">Kommunikationsabbruch:</strong> Das Display erkennt dies und stoppt abrupt das Senden von TX-Paketen.</li>
                           <li><strong class="font-medium text-slate-900">Controller-Timeout:</strong> Der Motorcontroller empfängt keine Daten mehr. Nach einem internen Timeout (ca. 200-500 ms) schaltet er sich ebenfalls in den Standby-Modus.</li>
                        </ol>
                    </div>
                </div>
            </section>

            <!-- Modes Section -->
            <section id="modes" class="content-section">
                <h1 class="text-3xl font-bold mb-2 text-slate-900">Referenztabelle der Fahrmodi</h1>
                <p class="mb-8 text-slate-600">Vergleich der Einstellungen für die verschiedenen Geschwindigkeitsstufen.</p>
                <div class="bg-white rounded-xl shadow-sm border border-slate-100 overflow-hidden">
                    <div class="overflow-x-auto">
                        <table class="w-full text-sm text-left text-slate-500">
                            <thead class="text-xs text-slate-700 uppercase bg-slate-100">
                                <tr>
                                    <th scope="col" class="px-6 py-3">Profil (TX 0x0C)</th>
                                    <th scope="col" class="px-6 py-3">Fahrmodus (TX 0x04)</th>
                                    <th scope="col" class="px-6 py-3">Max. Throttle (TX 0x10)</th>
                                    <th scope="col" class="px-6 py-3">Min. Speed-Wert (RX 0x08)</th>
                                    <th scope="col" class="px-6 py-3">V-Max (ca.)</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr class="bg-white border-b hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Begrenzt (`0x0C19`)</td>
                                    <td class="px-6 py-4">Stufe 1 (`0x05`)</td>
                                    <td class="px-6 py-4 font-mono">`0x0168` (360)</td>
                                    <td class="px-6 py-4 font-mono">`~0x00A9` (169)</td>
                                    <td class="px-6 py-4">15 km/h</td>
                                </tr>
                                <tr class="bg-white border-b hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Begrenzt (`0x0C19`)</td>
                                    <td class="px-6 py-4">Stufe 2 (`0x0A`)</td>
                                    <td class="px-6 py-4 font-mono">`0x02A8` (680)</td>
                                    <td class="px-6 py-4 font-mono">`~0x0080` (128)</td>
                                    <td class="px-6 py-4">20 km/h</td>
                                </tr>
                                <tr class="bg-white border-b hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Begrenzt (`0x0C19`)</td>
                                    <td class="px-6 py-4">Stufe 3 (`0x0F`)</td>
                                    <td class="px-6 py-4 font-mono">`0x03E8` (1000)</td>
                                    <td class="px-6 py-4 font-mono">`~0x0067` (103)</td>
                                    <td class="px-6 py-4">25 km/h</td>
                                </tr>
                                <tr class="bg-white border-b hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Offen (`0x0C64`)</td>
                                    <td class="px-6 py-4">Stufe 1 (`0x05`)</td>
                                    <td class="px-6 py-4 font-mono">`0x0168` (360)</td>
                                    <td class="px-6 py-4 font-mono">`~0x0086` (134)</td>
                                    <td class="px-6 py-4">19 km/h</td>
                                </tr>
                                <tr class="bg-white border-b hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Offen (`0x0C64`)</td>
                                    <td class="px-6 py-4">Stufe 2 (`0x0A`)</td>
                                    <td class="px-6 py-4 font-mono">`0x02A8` (680)</td>
                                    <td class="px-6 py-4 font-mono">`~0x0043` (67)</td>
                                    <td class="px-6 py-4">38 km/h</td>
                                </tr>
                                <tr class="bg-white hover:bg-slate-50">
                                    <td class="px-6 py-4 font-mono">Offen (`0x0C64`)</td>
                                    <td class="px-6 py-4">Stufe 3 (`0x0F`)</td>
                                    <td class="px-6 py-4 font-mono">`0x03E8` (1000)</td>
                                    <td class="px-6 py-4 font-mono">`~0x0030` (48)</td>
                                    <td class="px-6 py-4">53 km/h</td>
                                </tr>
                            </tbody>
                        </table>
                    </div>
                </div>
            </section>
        </main>
    </div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    
    const txPacketData = [
        { offset: 0, len: 1, name: 'Start-Marker', value: '01', desc: 'Feste Kennung für TX-Pakete.' },
        { offset: 1, len: 1, name: 'Paketlänge', value: '14', desc: 'Immer 20 Bytes (hex 14).' },
        { offset: 2, len: 2, name: 'Kommando-Typ', value: '0102', desc: 'Statischer Befehls-Header.' },
        { offset: 4, len: 1, name: 'Fahrmodus', value: '05/0A/0F', desc: 'Wählt die Leistungsstufe (1/2/3).' },
        { offset: 5, len: 1, name: 'Funktions-Bitmaske', value: '80', desc: 'Steuert Zusatzfunktionen via Bits: +0x40(Bremse), +0x20(Licht), +0x10(Hupe), +0x08(Blinker L), +0x04(Blinker R).' },
        { offset: 6, len: 6, name: 'Statische Parameter', value: 'const', desc: 'Unveränderte Konfigurationswerte.' },
        { offset: 12, len: 2, name: 'Geschwindigkeits-Profil', value: '0C19/0C64', desc: 'Wählt die V-Max Begrenzung (Standard/Offen). Little-Endian.' },
        { offset: 14, len: 2, name: 'Statischer Parameter', value: '01A4', desc: 'Unveränderter Konfigurationswert.' },
        { offset: 16, len: 2, name: 'Throttle-Sollwert', value: '0000-03E8', desc: 'Gashebel-Position (Little-Endian). Maximalwert wird vom Fahrmodus skaliert.' },
        { offset: 18, len: 2, name: 'Checksumme', value: 'variiert', desc: 'CRC-16/MODBUS der ersten 18 Bytes (Little-Endian).' }
    ];

    const rxPacketData = [
        { offset: 0, len: 1, name: 'Start-Marker', value: '02', desc: 'Antwort-Kennung.' },
        { offset: 1, len: 1, name: 'Paketlänge', value: '0E', desc: '14 Bytes Nutzdaten.' },
        { offset: 2, len: 1, name: 'Status-Typ', value: '01', desc: 'Antworttyp.' },
        { offset: 3, len: 1, name: 'Status-Flag', value: '00/80', desc: '0x80 wird einmalig beim Start als "Boot-Acknowledged" gesendet. Sonst 0x00.' },
        { offset: 4, len: 1, name: 'System-Status', value: 'C0/E0', desc: 'Meldet den Bremsstatus. 0xC0 (Normal). 0xE0 wenn Bremse aktiv ist (Bit 5 wird gesetzt).' },
        { offset: 5, len: 3, name: 'Unbekannt', value: '000000', desc: 'Bisher konstant bei 0x000000.' },
        { offset: 8, len: 2, name: 'Geschwindigkeit', value: '0DAC (Idle)', desc: 'Der eigentliche Geschwindigkeitswert. 2-Byte Big-Endian, invers proportional zur Raddrehzahl.' },
        { offset: 10, len: 2, name: 'Unbekannt', value: '0000', desc: 'Unbekannt.' },
        { offset: 12, len: 1, name: 'Status Bremse/Reku', value: '6C/4C', desc: 'Wert ändert sich von 0x6C zu 0x4C, wenn die Bremse aktiv ist.' },
        { offset: 13, len: 1, name: 'Checksumme', value: 'variiert', desc: 'CRC-8 Prüfsumme der ersten 13 Bytes (0x00-0x0C).' },
        { offset: 14, len: 2, name: 'Echo/Prüfung', value: '020E', desc: 'Wiederholung des Paket-Headers (Start-Marker & Paketlänge).' }
    ];

    function populatePacketExplorer(containerId, detailsId, packetData, totalBytes) {
        const container = document.getElementById(containerId);
        const details = document.getElementById(detailsId);
        if (!container || !details) return;

        let byteIndex = 0;
        let fieldIndex = 0;

        while(byteIndex < totalBytes) {
            const field = packetData.find(f => f.offset === byteIndex);
            
            const len = field ? field.len : 1;
            
            for(let i = 0; i < len; i++) {
                const block = document.createElement('div');
                block.classList.add('byte-block', 'p-2', 'rounded-lg', 'bg-white', 'border', 'border-slate-200');
                block.dataset.fieldIndex = fieldIndex;
                
                const offsetEl = document.createElement('div');
                offsetEl.classList.add('byte-offset', 'text-xs', 'text-slate-400');
                offsetEl.textContent = `0x${(byteIndex + i).toString(16).toUpperCase().padStart(2, '0')}`;
                
                const valueEl = document.createElement('div');
                valueEl.classList.add('byte-value', 'font-bold', 'text-slate-800');
                valueEl.textContent = '??';
                
                block.appendChild(valueEl);
                block.appendChild(offsetEl);
                container.appendChild(block);
            }
            
            if (field) {
                fieldIndex++;
            }
            byteIndex += len;
        }

        container.addEventListener('mouseover', (e) => {
            const block = e.target.closest('.byte-block');
            if (block) {
                updateDetails(block.dataset.fieldIndex, detailsId, packetData, containerId);
            }
        });
        container.addEventListener('click', (e) => {
            const block = e.target.closest('.byte-block');
            if (block) {
                updateDetails(block.dataset.fieldIndex, detailsId, packetData, containerId);
            }
        });
    }

    function updateDetails(fieldIndex, detailsId, packetData, containerId) {
        const details = document.getElementById(detailsId);
        const container = document.getElementById(containerId);
        const field = packetData[fieldIndex];

        details.innerHTML = `
            <h3 class="text-lg font-semibold mb-2 text-slate-900">${field.name}</h3>
            <p class="text-slate-600 mb-3">${field.desc}</p>
            <div class="flex space-x-4 text-sm">
                <span class="font-mono"><strong>Offset:</strong> 0x${field.offset.toString(16).toUpperCase().padStart(2, '0')}</span>
                <span class="font-mono"><strong>Länge:</strong> ${field.len} Byte(s)</span>
                <span class="font-mono"><strong>Beispiel:</strong> ${field.value}</span>
            </div>
        `;
        
        container.querySelectorAll('.byte-block').forEach(b => b.classList.remove('highlight'));
        for(let i = 0; i < field.len; i++) {
            container.children[field.offset + i].classList.add('highlight');
        }
    }

    populatePacketExplorer('tx-packet-container', 'tx-packet-details', txPacketData, 20);
    populatePacketExplorer('rx-packet-container', 'rx-packet-details', rxPacketData, 16);
    
    // Navigation
    const navLinks = document.querySelectorAll('.nav-link');
    const contentSections = document.querySelectorAll('.content-section');

    document.getElementById('main-nav').addEventListener('click', (e) => {
        e.preventDefault();
        const targetId = e.target.closest('.nav-link')?.dataset.target;
        if (!targetId) return;

        navLinks.forEach(link => link.classList.remove('active'));
        e.target.closest('.nav-link').classList.add('active');

        contentSections.forEach(section => {
            if (section.id === targetId) {
                section.classList.add('active');
            } else {
                section.classList.remove('active');
            }
        });
    });

    // Checksum Calculators
    function hexStringToByteArray(hexString) {
        return hexString.replace(/\s/g, '').match(/.{1,2}/g)?.map(byte => parseInt(byte, 16)) || [];
    }

    document.getElementById('calculate-tx-checksum').addEventListener('click', () => {
        const input = document.getElementById('tx-checksum-input').value;
        const resultEl = document.getElementById('tx-checksum-result');
        const bytes = hexStringToByteArray(input);
        
        if (bytes.length !== 18) {
            resultEl.textContent = 'Fehler: Es müssen genau 18 Bytes sein.';
            resultEl.classList.add('text-red-600');
            return;
        }
        
        const crc = crc16modbus(bytes);
        const crcBytes = [(crc & 0xFF), (crc >> 8)];
        resultEl.textContent = `Hex: ${crcBytes[0].toString(16).toUpperCase().padStart(2, '0')} ${crcBytes[1].toString(16).toUpperCase().padStart(2, '0')} (Little-Endian)`;
        resultEl.classList.remove('text-red-600');
    });

    document.getElementById('calculate-rx-checksum').addEventListener('click', () => {
        const input = document.getElementById('rx-checksum-input').value;
        const resultEl = document.getElementById('rx-checksum-result');
        const bytes = hexStringToByteArray(input);

        if (bytes.length !== 13) {
            resultEl.textContent = 'Fehler: Es müssen genau 13 Bytes sein.';
            resultEl.classList.add('text-red-600');
            return;
        }

        const crc = crc8(bytes);
        resultEl.textContent = `Hex: ${crc.toString(16).toUpperCase().padStart(2, '0')}`;
        resultEl.classList.remove('text-red-600');
    });

    function crc16modbus(buffer) {
        let crc = 0xFFFF;
        for (let pos = 0; pos < buffer.length; pos++) {
            crc ^= buffer[pos];
            for (let i = 8; i !== 0; i--) {
                if ((crc & 0x0001) !== 0) {
                    crc >>= 1;
                    crc ^= 0xA001;
                } else {
                    crc >>= 1;
                }
            }
        }
        return crc;
    }
    
    function crc8(buffer) {
        let crc = 0x00;
        const poly = 0x31;
        for (let i = 0; i < buffer.length; i++) {
            crc ^= buffer[i];
            for (let j = 0; j < 8; j++) {
                if ((crc & 0x80) !== 0) {
                    crc = ((crc << 1) & 0xFF) ^ poly;
                } else {
                    crc = (crc << 1) & 0xFF;
                }
            }
        }
        return crc;
    }
});
</script>

</body>
</html>
