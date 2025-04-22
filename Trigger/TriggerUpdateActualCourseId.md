create function "TriggerUpdateActualCourseId"() returns trigger
    language plpgsql
as
$$
BEGIN
    -- Print the id of the payment being processed
    RAISE NOTICE 'Processing Payment with id: %, paymentForId: %, courseOfferId: %', NEW."id", NEW."paymentForId", NEW."courseOfferId";

    -- Set the actualCourseId based on the result from the function
    NEW."actualCourseId" := "DetermineActualCourseId"(NEW."paymentForId", NEW."courseOfferId");

    -- Print the updated actualCourseId
    RAISE NOTICE 'Updated actualCourseId to: %', NEW."actualCourseId";

    -- Return the updated record with the new actualCourseId
    RETURN NEW;
END;
$$;

alter function "TriggerUpdateActualCourseId"() owner to learner;
