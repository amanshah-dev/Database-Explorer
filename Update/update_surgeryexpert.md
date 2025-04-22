create procedure update_surgeryexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Surgery', achievementCategoryId, userIds);
END;
$$;

alter procedure update_surgeryexpert(bigint, bigint[]) owner to learner;
