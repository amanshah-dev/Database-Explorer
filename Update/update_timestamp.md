create function update_timestamp() returns trigger
    language plpgsql
as
$$
BEGIN
    NEW."updatedAt" = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;

alter function update_timestamp() owner to neetprep_rw;
