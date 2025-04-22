create function update_data_uid_fields() returns trigger
    language plpgsql
as
$$
BEGIN
    -- Check if details column has the specified properties
    IF (NEW.details->>'startDataUid') IS NOT NULL THEN
        NEW."startDataUid" := NEW.details->>'startDataUid';
    END IF;
    IF (NEW.details->>'endDataUid') IS NOT NULL THEN
        NEW."endDataUid" := NEW.details->>'endDataUid';
    END IF;

    RETURN NEW;
END;
$$;

alter function update_data_uid_fields() owner to neetprep_rw;
