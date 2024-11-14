# HospitalClient
Progetto di Databases 2 2024/2025 di Roggeri Andrea, matricola 1079033.
## Introduzione
Il progetto riguarda un sistema di gestione ospedaliero. Viene utilizzato un database PostgreSQL per i dati e un client da riga di comando per interfacciarsi col database.
## Requisiti
[PostgreSQL](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) installato. <br />
[Python](https://www.python.org/downloads/) installato.<br />
Psycopg2 Installato(comando da Terminale per l'installazione: ```pip install Flask psycopg2```). <br />
## Configurazione database
Innanzitutto è necessario creare un database in postgreSQL tramite la SQL Shell di PostgreSQL, assicurandosi prima di utilizzare la porta predefinita ```5432```, l'username predefinito ```postgres``` e la password ```admin```(configurabile al momento dell'installazione di PostgreSQL). <br />
Utilizzare il comando ```CREATE DATABASE hospital_management;``` nella Shell per creare il database, quindi connettersi ad esso tramite il comando ```\c hospital_management;```.<br />
Ora che si è connessi al database, aggiungere le tabelle, trigger e viste materializzate necessarie tramite i comandi contenuti nel file [Commands.md](https://github.com/a-roggeri/HospitalClient/blob/master/Commands.md).<br />
Infine popolare il database tramite gli insert contenuti nel file [Populate.md](https://github.com/a-roggeri/HospitalClient/blob/master/Populate.md).<br />

Se dovesse essere necessario, è possibile rimuovere tutte le tuple senza alterare la struttura del database tramite il comando ```TRUNCATE TABLE Pazienti, Personale, Cartelle_Cliniche, Appuntamenti, Prescrizioni, log_eventi, log_modifiche, storico_appuntamenti RESTART IDENTITY CASCADE;```.
