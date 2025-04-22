create function update_mistake_count() returns trigger
    language plpgsql
as
$$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE "Mistake"
        SET "count" = "count" + 1
        WHERE "id" = NEW."mistakeId" AND "deleted" = FALSE;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE "Mistake"
        SET "count" = "count" - 1
        WHERE "id" = OLD."mistakeId" AND "deleted" = FALSE;
    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD."mistakeId" <> NEW."mistakeId" THEN
            UPDATE "Mistake"
            SET "count" = "count" - 1
            WHERE "id" = OLD."mistakeId" AND "deleted" = FALSE;
            
            UPDATE "Mistake"
            SET "count" = "count" + 1
            WHERE "id" = NEW."mistakeId" AND "deleted" = FALSE;
        END IF;
    END IF;
    RETURN NULL;
END;
$$;

alter function update_mistake_count() owner to neetprep_rw;
