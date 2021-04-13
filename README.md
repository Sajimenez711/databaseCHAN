# databaseCHAN
Database creation exercise , consisting on creating a DB, with triggers for password encryption, views and backup jobs

List of instructions 

##Create user
CREATE USER myuser WITH PASSWORD 'password';
##Create Database
CREATE DATABASE challenge WITH OWNER postgres; 
##Create Tables
CREATE TABLE "User" (
	id uuid PRIMARY KEY,
	name varchar(50),
	email varchar(20),
	password varchar(100),
	"createdAt" timestamp DEFAULT current_timestamp,
	"updatedAt" timestamp DEFAULT current_timestamp
);

CREATE TABLE "Team" (
	id uuid PRIMARY KEY,
	name varchar(50),
	"createdBy" uuid,
	Constraint team_user_fkey
		FOREIGN KEY ("createdBy") REFERENCES "User"(id),
	"createdAt" timestamp DEFAULT current_timestamp,
    "updatedAt" timestamp DEFAULT current_timestamp
);

CREATE TABLE "Account" (
	id uuid PRIMARY KEY,
	name varchar(50),
	"clientName" varchar(50),
	"leaderName" varchar(50),
	team uuid,
	Constraint account_team_fkey
		FOREIGN KEY ("team") REFERENCES "Team"(id),
	"createdAt" timestamp DEFAULT current_timestamp,
	"updatedAt" timestamp DEFAULT current_timestamp
	
);

CREATE TABLE "UserTeam" (
	id uuid Primary Key,
	"userId" uuid,
	"teamId" uuid,
	"createdAt" timestamp DEFAULT current_timestamp,
    "updatedAt" timestamp DEFAULT current_timestamp,
	Constraint user_fk
		FOREIGN KEY("userId") REFERENCES "User"(id),
	Constraint team_fk
		FOREIGN KEY("teamId") REFERENCES "Team"(id)
);

CREATE TABLE "MovementLog" (
	id uuid Primary Key,
	"userId" uuid,
	"teamId" uuid,
	"movementType" varchar(10) NOT NULL check( "movementType" IN ('INSERT', 'DELETE')),
	"createdAt" timestamp DEFAULT current_timestamp,
    "updatedAt" timestamp DEFAULT current_timestamp,
	Constraint user_fk
		FOREIGN KEY("userId") REFERENCES "User"(id),
	Constraint team_fk
		FOREIGN KEY("teamId") REFERENCES "Team"(id)
);

##Create View 
CREATE VIEW "UserTeamHistory" AS 
SELECT * FROM public."UserTeam"

##Create Function/Tigger
CREATE OR REPLACE FUNCTION hash_password() RETURNS trigger AS 
$$ BEGIN
    IF tg_op ='INSERT' OR tg_op = 'UPDATE' THEN
    NEW.password = MD5(NEW.password || NEW.id);
            RETURN NEW;
        END IF;
     END;
     $$ LANGUAGE plpgsql;
     CREATE TRIGGER password_hash
     BEFORE INSERT OR UPDATE Of password ON public."User" 
     FOR EACH ROW EXECUTE PROCEDURE hash_password();

##Create Log table procedure
CREATE OR REPLACE FUNCTION if_modified_func() RETURNS trigger AS $body$
DECLARE
    v_old_data JSON;
	v_new_data JSON;
BEGIN
    if (TG_OP = 'DELETE') then
        v_old_data := row_to_json(OLD);
        insert into public."MovementLog" (id, "userId", "teamId", "movementType")
        values (md5(random()::text || clock_timestamp()::text)::uuid, v_old_data->>'userId', v_old_data->>'teamId', 'DELETE');
        RETURN OLD;
    elsif (TG_OP = 'INSERT') then
		v_new_data := row_to_json(NEW);
		raise notice 'Value: %', TG_OP;
        insert into public."MovementLog" (id, "userId", "teamId", "movementType")
        values (md5(random()::text || clock_timestamp()::text)::uuid, (v_new_data->>'userId')::uuid, (v_new_data->>'teamId')::uuid, 'INSERT');
        RETURN NEW;
    else
        RAISE WARNING '[IF_MODIFIED_FUNC] - Other action occurred: %, at %',TG_OP,now();
        RETURN NULL;
    end if;

EXCEPTION
    WHEN data_exception THEN
        RAISE WARNING '[IF_MODIFIED_FUNC] - UDF ERROR [DATA EXCEPTION] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
    WHEN unique_violation THEN
        RAISE WARNING '[IF_MODIFIED_FUNC] - UDF ERROR [UNIQUE] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
    WHEN others THEN
        RAISE WARNING '[IF_MODIFIED_FUNC] - UDF ERROR [OTHER] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
END;
$body$
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = pg_catalog, audit;

DROP TRIGGER IF EXISTS movement_log_trigger ON "UserTeam";
CREATE TRIGGER movement_log_trigger
AFTER INSERT OR UPDATE OR DELETE ON "UserTeam"
FOR EACH ROW EXECUTE PROCEDURE if_modified_func();


##Create Backup Job in terminal 
##Note: this works in Unix/Linux based OS, if you are using something differente you can use the pgAgent extension of pgAdmin 
crontab -e 

## This will mke the backup every day at 00:00
00 00 * * * /usr/local/bin/pg_dump -U postgres challenge > backup.sql

##create
 .pgpass file
 
## pgpass format

hostname:port:database:username:password

 ##give it permisssions
 chmod 600 .pgpass

export PGPASSFILE=~/.pgpass

## for email handling use pgMail
