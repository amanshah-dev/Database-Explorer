create procedure update_ophthalmologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Ophthalmology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_ophthalmologyexpert(bigint, bigint[]) owner to learner;
