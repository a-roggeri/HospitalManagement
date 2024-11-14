CREATE TABLE Pazienti (
    ID SERIAL PRIMARY KEY,
    nome VARCHAR(50) NOT NULL,
    cognome VARCHAR(50) NOT NULL,
    data_di_nascita DATE NOT NULL,
    sesso CHAR(1) CHECK (sesso IN ('M', 'F', 'O')) NOT NULL,
    indirizzo TEXT NOT NULL,
    telefono VARCHAR(15) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    data_registrazione TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Personale (
    ID SERIAL PRIMARY KEY,
    nome VARCHAR(50) NOT NULL,
    cognome VARCHAR(50) NOT NULL,
    ruolo VARCHAR(50) CHECK (ruolo IN ('ADMIN', 'MEDICO', 'INFERMIERE', 'RECEPTIONIST')) NOT NULL,
    reparto VARCHAR(50) NOT NULL,
    telefono VARCHAR(15) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    data_assunzione DATE NOT NULL
);

CREATE TABLE Cartelle_Cliniche (
    ID SERIAL PRIMARY KEY,
    paziente_id INT NOT NULL REFERENCES Pazienti(ID) ON DELETE CASCADE,
    data_creazione TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    note_mediche TEXT,
    stato_attuale VARCHAR(100) CHECK (stato_attuale IN ('RICOVERATO', 'AMBULATORIALE', 'OSSERVAZIONE', 'DIMESSO')) NOT NULL
);

CREATE TABLE Appuntamenti (
    ID SERIAL PRIMARY KEY,
    paziente_id INT NOT NULL REFERENCES Pazienti(ID) ON DELETE CASCADE,
    personale_id INT NOT NULL REFERENCES Personale(ID) ON DELETE SET NULL,
    data DATE NOT NULL,
    ora TIME NOT NULL,
    motivo TEXT NOT NULL,
    stato VARCHAR(20) CHECK (stato IN ('PROGRAMMATO', 'COMPLETATO', 'ANNULLATO')) DEFAULT 'PROGRAMMATO' NOT NULL
);

CREATE TABLE Prescrizioni (
    ID SERIAL PRIMARY KEY,
    cartella_clinica_id INT NOT NULL REFERENCES Cartelle_Cliniche(ID) ON DELETE CASCADE,
    farmaco VARCHAR(100) NOT NULL,
    dosaggio VARCHAR(50) NOT NULL,
    frequenza VARCHAR(50) NOT NULL,
    durata INT,
    note TEXT
);

CREATE FUNCTION report_settimanale_appuntamenti()
RETURNS TABLE(personale_nome VARCHAR, personale_cognome VARCHAR, conteggio_appuntamenti INTEGER) AS $$
BEGIN
    RETURN QUERY
    SELECT p.nome, p.cognome, COUNT(a.ID)::INTEGER
    FROM Personale p
    JOIN Appuntamenti a ON p.ID = a.personale_id
    WHERE a.data >= NOW() - INTERVAL '7 days'
    GROUP BY p.nome, p.cognome;
END;
$$ LANGUAGE plpgsql;

CREATE FUNCTION log_nuovo_appuntamento()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO log_eventi (evento, descrizione, data_evento)
    VALUES ('Nuovo Appuntamento', 
            'Nuovo appuntamento per il paziente ID: ' || NEW.paziente_id || ' con il personale ID: ' || NEW.personale_id, 
            NOW());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE log_eventi (
    ID SERIAL PRIMARY KEY,
    evento VARCHAR(50),
    descrizione TEXT,
    data_evento TIMESTAMP
);

CREATE TRIGGER trigger_nuovo_appuntamento
AFTER INSERT ON Appuntamenti
FOR EACH ROW
EXECUTE FUNCTION log_nuovo_appuntamento();

CREATE MATERIALIZED VIEW vista_ricoveri_settimanali AS
SELECT paziente_id, COUNT(*) AS numero_ricoveri
FROM Cartelle_Cliniche
WHERE stato_attuale = 'RICOVERATO'
AND data_creazione >= NOW() - INTERVAL '7 days'
GROUP BY paziente_id;

CREATE TABLE log_modifiche (
    ID SERIAL PRIMARY KEY,
    cartella_clinica_id INT NOT NULL REFERENCES Cartelle_Cliniche(ID) ON DELETE CASCADE,
    modifica TEXT,
    data_modifica TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION log_modifica_cartella()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO log_modifiche (cartella_clinica_id, modifica)
    VALUES (NEW.ID, 'Modifica alle note_mediche o stato_attuale');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_modifica_cartella
AFTER UPDATE ON Cartelle_Cliniche
FOR EACH ROW
EXECUTE FUNCTION log_modifica_cartella();

CREATE TABLE storico_appuntamenti (
    LIKE Appuntamenti INCLUDING ALL
);

CREATE OR REPLACE FUNCTION sposta_appuntamento_storico()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.stato = 'COMPLETATO' THEN
        INSERT INTO storico_appuntamenti SELECT * FROM Appuntamenti WHERE ID = NEW.ID;
        DELETE FROM Appuntamenti WHERE ID = NEW.ID;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_sposta_appuntamento_storico
AFTER UPDATE OF stato ON Appuntamenti
FOR EACH ROW
WHEN (NEW.stato = 'COMPLETATO')
EXECUTE FUNCTION sposta_appuntamento_storico();

CREATE MATERIALIZED VIEW analisi_mensile_pazienti AS
SELECT
    DATE_TRUNC('month', data_registrazione) AS mese,
    COUNT(ID) AS numero_pazienti
FROM Pazienti
GROUP BY mese
ORDER BY mese DESC;

CREATE INDEX idx_fts_note_mediche ON Cartelle_Cliniche
USING gin(to_tsvector('italian', note_mediche));