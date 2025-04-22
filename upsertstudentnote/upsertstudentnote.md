create function upsertstudentnote(_userid integer, _questionid integer, _studentattachimguri text, _note character varying)
    returns TABLE(studentnoteid bigint, userid integer, questionid integer, studentattachimguri text, notetext character varying, createdat timestamp without time zone, updatedat timestamp without time zone)
    language plpgsql
as
$$
DECLARE
    existing_note_id BIGINT;
BEGIN
    -- Attempt to find an existing note by questionId with explicit column reference
    SELECT "id" INTO existing_note_id FROM "StudentNote" WHERE "StudentNote"."questionId" = _questionId;

    IF existing_note_id IS NULL THEN
        -- No existing note, perform an insert
        RETURN QUERY INSERT INTO "StudentNote" (
            "userId", "questionId", "studentAttachImgUri", "note", "createdAt", "updatedAt"
        )
        VALUES (_userId, _questionId, _studentAttachImgUri, _note, now(), now())
        RETURNING "id" AS studentNoteId, "userId", "questionId", "studentAttachImgUri", "note" AS noteText, "createdAt", "updatedAt";
    ELSE
        -- Existing note found, perform an update
        RETURN QUERY UPDATE "StudentNote"
        SET 
            "userId" = _userId,
            "studentAttachImgUri" = _studentAttachImgUri,
            "note" = _note,
            "updatedAt" = now()
        WHERE "StudentNote"."id" = existing_note_id
        RETURNING "id" AS studentNoteId, "userId", "questionId", "studentAttachImgUri", "note" AS noteText, "createdAt", "updatedAt";
    END IF;
END;
$$;

alter function upsertstudentnote(integer, integer, text, varchar) owner to neetprep_rw;
