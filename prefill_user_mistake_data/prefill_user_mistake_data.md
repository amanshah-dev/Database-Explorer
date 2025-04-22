create function prefill_user_mistake_data() returns trigger
    language plpgsql
as
$$
BEGIN
    SELECT "content" INTO NEW."content"
    FROM "Mistake"
    WHERE "id" = NEW."mistakeId";
    RETURN NEW;
END;
$$;

alter function prefill_user_mistake_data() owner to neetprep_rw;
