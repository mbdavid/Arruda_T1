# Infraestrutura para Gestão de Dados

#### Trabalho 2

Maurício David

28/10/2024

---

## Descrição das consultas

### Q1 -> Q2:

- Q1: Dado um código de aeroporto IATA, listar todos os voos que partem deste aeroporto
- Q2: Para cada voo obtido em Q1, recuperar o nome dos passageiros e o numero do acento pelo numero de voo

### Q3->Q4

- Q3: Dado um determinado pais e cidade, listas todos os aeroportos que possuem base nesta pais
- Q4: Para cada voo companhia obtida em Q1, recuperar todas as aeronaves e suas capacidades de acentos

## Comandos DDL (Data Definition Language)

Definição de keyspace e tabelas para o caso Q1 -> Q2

```sql
CREATE KEYSPACE airline_1 WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 1};

CREATE TABLE airline_1.flights_by_departure_airport (
    departure_airport_iata text,
    flight_id bigint,
    flightno text,
    airline_id bigint,
    to_airport_iata text,
    airplane_id bigint,
    departure timestamp,
    arrival timestamp,
    PRIMARY KEY (departure_airport_iata, flight_id)
);

CREATE TABLE airline_1.passengers_by_flight (
    flight_id bigint,
    passenger_id bigint,
    seat text,
    firstname text,
    lastname text,
    PRIMARY KEY (flight_id, seat)
);
```

Definição de tabelas para o caso Q3 -> Q4

```sql
CREATE KEYSPACE airline_2 WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 1};

CREATE TABLE airline_2.base_airports_by_country_city (
    country text,
    city text,
    airport_id int,
    airport_name text,
    airline_id int,
    airline_name text,
    PRIMARY KEY ((country, city), airport_id, airline_id)
);

CREATE TABLE airline_2.airplanes_by_airline (
    airline_id int,
    airplane_id int,
    airplane_type_id int,
    airplane_type_name text,
    capacity int,
    PRIMARY KEY (airline_id, airplane_id)
);
```

## Comandos CQL DML (Data Manipulation Language)

Para a inclusão dos dados vindo o banco relacional para o Cassandra, as seguintes consultas foram realizadas:

### Inclusão para Q1->Q2

```sql
-- Consulta no banco relacional
SELECT
    'INSERT INTO airline_1.flights_by_departure_airport (departure_airport_iata, flight_id, flightno, airline_id, to_airport_iata, airplane_id, departure, arrival) VALUES ('''
    + dep_airport.IATA + ''', '
    + CAST(f.FLIGHT_ID AS VARCHAR(20)) + ', '''
    + f.FLIGHTNO + ''', '
    + CAST(f.AIRLINE_ID AS VARCHAR(20)) + ', '''
    + arr_airport.IATA + ''', '
    + CAST(f.AIRPLANE_ID AS VARCHAR(20)) + ', '''
    + CONVERT(VARCHAR(30), f.DEPARTURE, 126) + ''', '''
    + CONVERT(VARCHAR(30), f.ARRIVAL, 126) + ''');'
FROM
    AIR_FLIGHTS f
JOIN
    AIR_AIRPORTS dep_airport ON f.FROM_AIRPORT_ID = dep_airport.AIRPORT_ID
JOIN
    AIR_AIRPORTS arr_airport ON f.TO_AIRPORT_ID = arr_airport.AIRPORT_ID
WHERE dep_airport.IATA = 'GRU';

-- Insert gerado para o Cassandra
INSERT INTO airline_1.flights_by_departure_airport (departure_airport_iata, flight_id, flightno, airline_id, to_airport_iata, airplane_id, departure, arrival) VALUES ('GRU', 389297, 'TU9679  ', 99, 'CLN', 3610, '2023-11-19T15:20:56', '2023-02-18T17:58:00');

-- Consulta no banco relacional
SELECT
    'INSERT INTO airline_1.passengers_by_flight (flight_id, passenger_id, seat, firstname, lastname) VALUES ('
    + CAST(b.FLIGHT_ID AS VARCHAR(20)) + ', '
    + CAST(b.PASSENGER_ID AS VARCHAR(20)) + ', '''
    + b.SEAT + ''', '''
    + p.FIRSTNAME + ''', '''
    + p.LASTNAME + ''');'
FROM
    AIR_BOOKINGS b
JOIN
    AIR_PASSENGERS p ON b.PASSENGER_ID = p.PASSENGER_ID
WHERE
    b.FLIGHT_ID = 389297
```

### Inclusão para Q3->Q4

```sql
-- Consulta no banco relacional
SELECT
    'INSERT INTO airline_2.base_airports_by_country_city (country, city, airport_id, airport_name, airline_id, airline_name) VALUES ('''
    + REPLACE(aag.COUNTRY, '''', '''''') + ''', '''
    + REPLACE(aag.CITY, '''', '''''') + ''', '
    + CAST(aa.AIRPORT_ID AS VARCHAR) + ', '''
    + REPLACE(aa.NAME, '''', '''''') + ''', '
    + CAST(al.AIRLINE_ID AS VARCHAR) + ', '''
    + REPLACE(al.AIRLINE_NAME, '''', '''''') + ''');'
FROM
    AIR_AIRLINES al
INNER JOIN AIR_AIRPORTS aa ON al.BASE_AIRPORT_ID = aa.AIRPORT_ID
INNER JOIN AIR_AIRPORTS_GEO aag ON aa.AIRPORT_ID = aag.AIRPORT_ID
WHERE
    aag.COUNTRY = 'BRAZIL'

-- Insert gerado para o Cassandra
INSERT INTO airline_2.base_airports_by_country_city (country, city, airport_id, airport_name, airline_id, airline_name) VALUES ('BRAZIL', 'NOVA IGUACU', 93, 'AEROCLUB', 12, 'Brazil Airlines');

-- Consulta no banco relacional
SELECT
    'INSERT INTO airline_2.airplanes_by_airline (airline_id, airplane_id, airplane_type_id, airplane_type_name, capacity) VALUES ('
    + CAST(aa.AIRLINE_ID AS VARCHAR) + ', '
    + CAST(aa.AIRPLANE_ID AS VARCHAR) + ', '
    + CAST(aa.AIRPLANE_TYPE_ID AS VARCHAR) + ', '''
    + REPLACE(at.NAME, '''', '''''') + ''', '
    + CAST(aa.CAPACITY AS VARCHAR) + ');'
FROM
    AIR_AIRPLANES aa
INNER JOIN AIR_AIRPLANE_TYPES at ON aa.AIRPLANE_TYPE_ID = at.AIRPLANE_TYPE_ID
WHERE aa.AIRLINE_ID = 93

-- Insert gerado para o Cassandra
INSERT INTO airline_2.airplanes_by_airline (airline_id, airplane_id, airplane_type_id, airplane_type_name, capacity) VALUES (93, 5038, 6, 'Airbus A380', 644);
... (+48 registros)
```

## Comandos DQL (Data Query Language)

### Consulta para Q1->Q2

```sql
-- Q1
SELECT *
  FROM airline_1.flights_by_departure_airport
 WHERE departure_airport_iata = 'GRU';
```

| departure_airport_iata | flight_id | airline_id | airplane_id | arrival                         | departure                       | flightno | to_airport_iata |
| ---------------------- | --------- | ---------- | ----------- | ------------------------------- | ------------------------------- | -------- | --------------- |
| GRU                    | 389297    | 99         | 3610        | 2023-02-18 17:58:00.000000+0000 | 2023-11-19 15:20:56.000000+0000 | TU9679   | CLN             |

```sql
-- Q2
SELECT firstname, lastname, seat
  FROM airline_1.passengers_by_flight
WHERE flight_id = 389297;
```

| firstname     | lastname   | seat |
| ------------- | ---------- | ---- |
| Todd          | Strasser   | 1A   |
| Krzysztof     | Pieczynski | 1C   |
| Clarence      | Thompson   | 1D   |
| Scott         | Rudolph    | 1F   |
| Jessica       | Platek     | 1G   |
| Romain        | Redler     | 2E   |
| Sandy ,       | V          | 2F   |
| Paul          | Young      | 3C   |
| James         | Brewer     | 3D   |
| John          | Tobias     | 3H   |
| Keith         | Lockhart   | 4B   |
| Chris         | Morris     | 4F   |
| Michael Keith | Aleshire   | 4H   |
| Tyler         | Yates      | 5A   |
| Aaron         | Bailey     | 5D   |
| Peter         | Sweval     | 6C   |
| Rose          | Byrne      | 6D   |
| Larry         | Little     | 6E   |
| Aleksandar    | Radojevic  | 6F   |
| Mike          | Hurlbut    | 7D   |
| Kenny         | Kelly      | 7G   |
| Pam           | Shriver    | 8B   |
| Junior        | Wells      | 8E   |
| Nancy         | Spungen    | 8G   |
| Brian         | Sikorski   | 9C   |

### Consulta para Q3->Q4

```sql
-- Q3
SELECT airport_id, airport_name, airline_id, airline_name
FROM airline_2.base_airports_by_country_city
WHERE country = 'BRAZIL' AND city = 'NOVA IGUACU';
```

| airport_id | airport_name | airline_id | airline_name    |
| ---------- | ------------ | ---------- | --------------- |
| 93         | AEROCLUB     | 12         | Brazil Airlines |

```sql
-- Q4
SELECT airplane_id, airplane_type_name, capacity
  FROM airline_2.airplanes_by_airline
 WHERE airline_id = 93;
```

| airplane_id | airplane_type_name      | capacity |
| ----------- | ----------------------- | -------- |
| 5026        | Boeing 737              | 114      |
| 5027        | Bombardier Q Series     | 78       |
| 5028        | Fokker 70               | 79       |
| 5029        | Fokker 70               | 79       |
| 5030        | Boeing 747              | 335      |
| 5031        | Boeing 737              | 114      |
| 5032        | Boeing 747              | 335      |
| 5033        | Boeing 767              | 200      |
| 5034        | Boeing 737              | 114      |
| 5035        | McDonnell Douglas DC-10 | 380      |
| 5036        | McDonnell Douglas DC-10 | 380      |
| 5037        | Bombardier Q Series     | 78       |
| 5038        | Airbus A380             | 644      |
| 5039        | Airbus A380             | 644      |
| 5040        | Boeing 737              | 114      |
| 5041        | Bombardier Q Series     | 78       |
| 5042        | Boeing 737              | 114      |
| 5043        | Boeing 777              | 420      |
| 5044        | Boeing 777              | 420      |
| 5045        | Airbus A380             | 644      |
| 5046        | Airbus A380             | 644      |
| 5047        | Airbus A330             | 420      |
| 5048        | Bombardier Q Series     | 78       |
| 5049        | Airbus A380             | 644      |
| 5050        | Boeing 737              | 114      |
| 5051        | Boeing 747              | 335      |
| 5052        | Airbus-A320-Familie     | 150      |
| 5053        | Douglas DC-9            | 115      |
| 5054        | Boeing 767              | 200      |
| 5055        | Douglas DC-9            | 115      |
| 5056        | Embraer-ERJ-145-Familie | 50       |
| 5057        | Airbus A380             | 644      |
| 5058        | Fokker 100              | 95       |
| 5059        | Boeing 767              | 200      |
