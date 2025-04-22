create function "PopulateDailyUserAllEvent"(days integer, eventdate date) returns void
    language plpgsql
as
$$
BEGIN
    PERFORM "PopulateDailyUserEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserTestQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserNcertQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserCorrectQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserPhysicsQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserChemistryQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserPhysicsCorrectQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserChemistryCorrectQEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserTestSubmitEvent"(days, date(eventDate));
    PERFORM "PopulateDailyUserVideoViewEvent"(days, date(eventDate));
END
$$;

alter function "PopulateDailyUserAllEvent"(integer, date) owner to learner;
